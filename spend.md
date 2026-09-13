# Spend Revenue Allocation

The query solves two separate “one thing maps to several rows” problems:

1. One tracking tag or ad name can map to multiple ad IDs.
2. One ad ID can appear in multiple physical spend rows.

These require two different weights:

```text
Revenue token
     |
     | allocation_weight
     v
Choose ad ID(s)
     |
     | ad_row_weight
     v
Choose physical output row(s)
```

## What is `ad_row_weight`?

`ad_row_weight` prevents revenue from being duplicated when the same ad appears in more than one spend row.

Revenue is matched to an ad ID, but the final report contains spend rows. Usually, an ad has one spend row per day:

| Ad ID | Ad name | Clicks |
| ---: | --- | ---: |
| 123 | summer-sale | 100 |

If ad 123 earned $50, the result is straightforward:

| Ad ID | Ad name | Revenue |
| ---: | --- | ---: |
| 123 | summer-sale | $50 |

But sometimes the same ad ID appears in multiple spend rows:

| Ad ID | Ad name | Clicks |
| ---: | --- | ---: |
| 123 | summer-sale | 60 |
| 123 | renamed-summer-sale | 40 |

Because the revenue join is based on ad 123, both rows match the same $50. Without protection, the result would be duplicated:

| Ad ID | Ad name | Revenue |
| ---: | --- | ---: |
| 123 | summer-sale | $50 |
| 123 | renamed-summer-sale | $50 |
|  | **Incorrect total** | **$100** |

`ad_row_weight` decides how much of the ad’s $50 belongs on each physical row. Because the rows have 60 and 40 clicks, their weights are 60% and 40%:

| Ad ID | Ad name | Clicks | `ad_row_weight` | Revenue |
| ---: | --- | ---: | ---: | ---: |
| 123 | summer-sale | 60 | 0.60 | $30 |
| 123 | renamed-summer-sale | 40 | 0.40 | $20 |
|  |  |  | **1.00** | **$50** |

If all rows have zero clicks, the query divides the weight evenly.

## 1. `ad_spend`: the input rows

[`ad_spend.sql:20`](devops/data/queries/boost_performance/ad_spend.sql#L20) starts with the existing daily Meta spend data and attaches the owning Wayward `account_id`.

A simplified row looks like this:

| Date | Account | `ext_ad_id` | Ad name | Clicks | Spend |
| --- | --- | ---: | --- | ---: | ---: |
| Aug 26 | Acme | 123 | summer-sale | 60 | $40 |

This is both:

- The source of ad activity: spend, clicks, and impressions.
- A lookup table that tells us which tracking tokens belong to which Meta ad IDs.

## 2. `spend`: add `ad_row_weight`

The first CTE, [`ad_performance_live.sql:5`](devops/data/queries/boost_performance/ad_performance_live.sql#L5), keeps every spend row and adds `ad_row_weight`.

Why is that necessary? The same ad ID can have multiple physical spend rows on the same day—for example, because of ad-name aliases or other dimensions:

| `ext_ad_id` | Ad name | Clicks |
| ---: | --- | ---: |
| 123 | summer-sale | 60 |
| 123 | renamed-summer-sale | 40 |

Revenue is eventually calculated at the stable ad-ID level:

> Ad 123 earned $100.

However, the output still contains both physical spend rows. If we put $100 on each row, the dashboard would show $200.

`ad_row_weight` divides the ad’s revenue between those rows using their clicks:

| `ext_ad_id` | Ad name | Clicks | `ad_row_weight` | Assigned revenue |
| ---: | --- | ---: | ---: | ---: |
| 123 | summer-sale | 60 | 0.60 | $60 |
| 123 | renamed-summer-sale | 40 | 0.40 | $40 |
|  |  |  | **1.00** | **$100** |

If all rows have zero clicks, the query splits the weight evenly instead.

The weight is applied when the final ad rows are created in [`ad_performance_live.sql:289`](devops/data/queries/boost_performance/ad_performance_live.sql#L289).

## 3. `spend_name_ads`: build the ad-name lookup

[`ad_performance_live.sql:21`](devops/data/queries/boost_performance/ad_performance_live.sql#L21) turns the spend rows into a lookup:

```text
(date, account, ad_name) → ext_ad_id
```

For example:

| Date | Account | Token | `ext_ad_id` | Clicks |
| --- | --- | --- | ---: | ---: |
| Aug 26 | Acme | summer-sale | 123 | 80 |

Here, `token` is simply `ad_name` renamed because it will be compared with revenue’s `utm_term`.

It groups repeated physical rows for the same `(date, account, ad_name, ad ID)` and sums their clicks. This prevents duplicate spend rows from creating duplicate lookup entries.

## 4. `spend_name_tokens`: handle shared ad names

The difficult case is that an ad name is not guaranteed to identify exactly one ad ID.

For example:

| Token | `ext_ad_id` | Clicks |
| --- | ---: | ---: |
| summer-sale | 123 | 80 |
| summer-sale | 456 | 20 |

Suppose an affiliate sale arrives with:

```text
match_kind = session_tag
utm_term   = summer-sale
revenue    = $100
```

We know it belongs to `summer-sale`, but that name identifies two ads. We cannot safely give both ads the full $100.

[`ad_performance_live.sql:33`](devops/data/queries/boost_performance/ad_performance_live.sql#L33) calculates an `allocation_weight` based on clicks:

| Token | `ext_ad_id` | `allocation_weight` | Assigned revenue |
| --- | ---: | ---: | ---: |
| summer-sale | 123 | 0.80 | $80 |
| summer-sale | 456 | 0.20 | $20 |
|  |  | **1.00** | **$100** |

If neither ad has clicks, it splits the revenue evenly.

This is the first fan-out protection:

> `allocation_weight` divides one revenue token among multiple ad IDs.

## 5. `spend_id_tokens`: the direct, unambiguous path

Some revenue already carries a stable external ad ID:

```text
match_kind = ad_id
utm_term   = 123
```

[`ad_performance_live.sql:49`](devops/data/queries/boost_performance/ad_performance_live.sql#L49) creates the direct lookup:

```text
(date, account, "123") → ad 123
```

Its `allocation_weight` is always 1 because the stable ad ID identifies the ad directly.

The `DISTINCT` matters because ad 123 may have several physical spend rows. We still want only one token-to-ad mapping:

| Match kind | Token | `ext_ad_id` | `allocation_weight` |
| --- | --- | ---: | ---: |
| ad_id | 123 | 123 | 1.00 |

## 6. `spend_tokens`: combine both lookup methods

[`ad_performance_live.sql:61`](devops/data/queries/boost_performance/ad_performance_live.sql#L61) combines:

- `session_tag` tokens matched through `ad_name`.
- `ad_id` tokens matched directly through `ext_ad_id`.

They remain typed with `match_kind`, so a tag whose text happens to be `123` cannot accidentally match ad ID 123.

Conceptually:

| Match kind | Token | `ext_ad_id` | `allocation_weight` |
| --- | --- | ---: | ---: |
| session_tag | summer-sale | 123 | 0.80 |
| session_tag | summer-sale | 456 | 0.20 |
| ad_id | 123 | 123 | 1.00 |
| ad_id | 456 | 456 | 1.00 |

## Complete example

Suppose `summer-sale` revenue is $100 and maps to:

- Ad 123 with 80% of clicks.
- Ad 456 with 20% of clicks.

First, `allocation_weight` divides revenue between ads:

```text
$100
├── Ad 123: $80
└── Ad 456: $20
```

Then suppose ad 123 has two physical spend rows. `ad_row_weight` divides its $80 across those output rows:

```text
Ad 123: $80
├── Spend row A: $48
└── Spend row B: $32

Ad 456: $20
└── Spend row C: $20
```

The final total remains exactly $100.

In short:

- `allocation_weight` divides revenue between different ads sharing the same tracking token.
- `ad_row_weight` divides an ad’s already-allocated revenue between multiple spend rows representing that same ad.

The full flow is also documented in [`README.md:67`](devops/data/queries/boost_performance/README.md#L67).
