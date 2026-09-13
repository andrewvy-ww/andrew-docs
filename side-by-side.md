# WAY-2234: Old versus ID-first spend/revenue joining

Status: **explanation only; untracked review draft**

This document compares only the part of Boost reporting that combines already
available Meta spend with already normalized affiliate revenue.

Out of scope:

- fetching Meta reports;
- fetching affiliate reports;
- building the fetchers event log; and
- calculating source deltas.

The question here starts after those steps:

```text
daily Meta spend rows + normalized affiliate revenue
                         ↓
               daily ad-performance rows
```

## Short version

The old method asks:

> Which spend row has a name or ID equal to this revenue tracking token?

The proposed method asks:

> Which stable ad ID did Layer B prove caused this revenue, and what is that
> ad's normalized daily spend?

```text
OLD
revenue token ──match name or ID──> one or more physical spend rows
                                      ↓
                              weights prevent duplication

NEW
revenue evidence ──resolve once──> stable ext_ad_id
                                      ↓
                         join one normalized ad/day row
```

## Inputs used in the examples

Assume Meta spend contains these physical rows for one account and day:

| Date | Account | `ext_ad_id` | Ad name | Clicks | Spend |
| --- | --- | --- | --- | ---: | ---: |
| Aug 26 | Acme | `123` | `summer-sale` | 60 | $30 |
| Aug 26 | Acme | `123` | `renamed-summer-sale` | 40 | $20 |
| Aug 26 | Acme | `456` | `summer-sale` | 20 | $10 |

There are two complications:

1. Ad `123` appears in two physical spend rows because its display name changed.
2. The name `summer-sale` is shared by two stable ads: `123` and `456`.

Assume normalized affiliate revenue contains these events:

| Event | Date | Account | Available evidence | Sales | Commission |
| --- | --- | --- | --- | ---: | ---: |
| A | Aug 26 | Acme | direct ad ID `123` | $50 | $5 |
| B | Aug 26 | Acme | session name `summer-sale`, no ad ID | $100 | $10 |
| C | Aug 29 | Acme | session ad ID `123`, no spend that day | $30 | $3 |
| D | Aug 26 | Popsugar | publisher tag `hyphenpopsugar-20` | $100 | $10 |

The examples below follow each event through both designs.

## 1. Old method: spend rows double as the attribution lookup

The old family includes the production canonical view and the paused weighted
Layer C migration. Their account-resolution details differ, but they share the
same fundamental shape: physical spend rows supply both the performance facts
and the tokens used to find an ad.

The relevant existing queries are:

- [`canonical/ad_spend_daily.sql`](../devops/data/queries/canonical/ad_spend_daily.sql)
  builds physical spend rows plus historical zero-spend name-ownership rows.
- [`canonical/ad_spend_and_affiliate_revenue_by_ad_by_date.sql`](../devops/data/queries/canonical/ad_spend_and_affiliate_revenue_by_ad_by_date.sql)
  is the legacy production ad-level join.
- [`boost_performance/ad_spend.sql`](../devops/data/queries/boost_performance/ad_spend.sql)
  overlays stable Wayward `account_id` on the inherited spend rows.
- [`boost_performance/ad_performance_live.sql`](../devops/data/queries/boost_performance/ad_performance_live.sql)
  is the weighted Layer C implementation being superseded by the ID-first plan.

### Step 1: turn each spend row into matching tokens

Each spend row can advertise two possible ways for revenue to find it:

```text
ad 123, name summer-sale
├── name token: summer-sale
└── ID token:   123
```

Our example becomes conceptually:

| Physical spend row | Match kind | Token | Candidate ad |
| --- | --- | --- | --- |
| `123 / summer-sale` | name/session tag | `summer-sale` | `123` |
| `123 / summer-sale` | ad ID | `123` | `123` |
| `123 / renamed-summer-sale` | name/session tag | `renamed-summer-sale` | `123` |
| `123 / renamed-summer-sale` | ad ID | `123` | `123` |
| `456 / summer-sale` | name/session tag | `summer-sale` | `456` |
| `456 / summer-sale` | ad ID | `456` | `456` |

In legacy production SQL, the condition is effectively:

```text
revenue.utm_term = spend.ad_name
OR
revenue.utm_term = spend.ext_ad_id
```

The paused Layer C work expresses the same choice as typed token rows so the
name `123` cannot accidentally match ad ID `123` merely because the strings are
equal.

### Step 2: join on date, account, and token

The old join requires a matching:

```text
date + account + tracking token
```

Legacy production derives the account-shaped key through `site` and account
display-name normalization. The paused migration improves that part by joining
on numeric `account_id`, but the spend-token attribution model remains.

### Step 3: use two weights to repair fan-out

Because spend rows double as the lookup, one revenue amount can match several
rows. Two separate weights are needed.

#### `allocation_weight`: one name identifies several ads

Event B carries `summer-sale` but no ad ID. That token identifies ads `123` and
`456`.

The weighted design uses clicks to estimate how much revenue belongs to each
ad. Across the matching name rows, ad `123` has 60 clicks and ad `456` has 20:

| Token | Candidate ad | Name-matched clicks | `allocation_weight` | $100 sales assigned |
| --- | --- | ---: | ---: | ---: |
| `summer-sale` | `123` | 60 | 75% | $75 |
| `summer-sale` | `456` | 20 | 25% | $25 |
| | | | **100%** | **$100** |

This conserves money, but it does not prove that 75% of the purchases came from
ad `123`. It is a deterministic estimate based on click share.

#### `ad_row_weight`: one ad appears in several physical spend rows

Event A directly identifies ad `123`, but ad `123` has two physical spend rows.
Without another protection, joining $50 to both rows would report $100.

The old weighted design divides ad `123`'s revenue across its two output rows by
their 60/40 clicks:

| Ad | Physical output row | Clicks | `ad_row_weight` | $50 sales assigned |
| --- | --- | ---: | ---: | ---: |
| `123` | `summer-sale` | 60 | 60% | $30 |
| `123` | `renamed-summer-sale` | 40 | 40% | $20 |
| | | | **100%** | **$50** |

Again, the weight prevents duplication. It exists because the output grain is
physical spend rows rather than one stable ad/day.

### Step 4: manufacture zero-spend name rows for delayed revenue

Affiliate conversions can arrive after an ad stops spending. The old spend view
extends a name's ownership for up to 30 days by creating zero-spend rows.

Conceptually, event C might find this generated row on Aug 29:

| Date | Account | Ad ID | Ad name | Spend | Why the row exists |
| --- | --- | --- | --- | ---: | --- |
| Aug 29 | Acme | `123` | `summer-sale` | $0 | 30-day ownership scaffold |

That lets a delayed name token continue finding an ad through a spend-shaped
row even though there was no actual spend that day.

For direct ad-ID revenue, the scaffold is conceptually unnecessary: the ad ID
already identifies the ad. It remains necessary in the old architecture because
the spend surface is also being used as the attribution phone book.

### Old result for the four example events

| Event | Old ad-level outcome |
| --- | --- |
| A: direct ID `123`, $50 | Matches ad `123`; `ad_row_weight` prevents its two spend rows from duplicating the $50. |
| B: shared name, $100 | Estimated across ads `123` and `456` by click share. |
| C: delayed ID `123`, $30 | Can match a generated zero-spend ownership row or another spend-shaped ID row, depending on the implementation. |
| D: publisher tag, $100 | No matching ad name or ID, so no ad-level match. Legacy account/day reporting still retains it under Popsugar through a separate site path. |

## 2. New method: resolve the ad first, then join stable facts

The proposed design is specified in
[`way-2234-id-first-boost-attribution.md`](./way-2234-id-first-boost-attribution.md).
It separates three jobs that the old spend rows combined:

```text
ad provenance    Which stable ads are Wayward ads, and which account owns them?
attribution      Which stable ad, if any, caused this revenue event?
daily spend      What did that stable ad spend on this date?
```

### Step 1: normalize spend to one stable ad/day row

The three physical input rows become two normalized facts:

| Date | Account | Platform | `ext_ad_id` | Clicks | Spend | Display name |
| --- | --- | --- | --- | ---: | ---: | --- |
| Aug 26 | Acme | Meta | `123` | 100 | $50 | chosen current/as-of name |
| Aug 26 | Acme | Meta | `456` | 20 | $10 | `summer-sale` |

The rename no longer creates two fact rows. Ad name is metadata, so changing it
cannot change the spend grain.

Because ad `123` now has one row, `ad_row_weight` disappears.

### Step 2: Layer B resolves each revenue event to one stable target

Before revenue reaches the spend join, Layer B applies typed evidence precedence:

```text
direct source ad ID
        ↓ otherwise
stable ad ID from the reader session
        ↓ otherwise
one unique historical name alias at the original click date
        ↓ otherwise
unproved exception
```

The resulting rows look like:

| Event | `attributed_ext_ad_id` | Method | Status | Sales |
| --- | --- | --- | --- | ---: |
| A | `123` | `direct_ad_id` | proven | $50 |
| B | NULL | `unproved` | exception: alias ambiguous | $100 |
| C | `123` | `session_ad_id` | proven | $30 |
| D | NULL | `unproved` | exception: publisher/account tag | $100 |

Event B becomes an exception because `summer-sale` identifies two ads. The new
method refuses to turn click proportions into commercial ad proof. Therefore
`allocation_weight` disappears from normal Boost attribution.

Event D remains visible for affiliate-source reconciliation and can remain
associated with Popsugar at the publisher/account level, but it is not included
in proven Boost ad performance.

### Step 3: create the complete set of stable ad/day output keys

The new performance view takes keys from both sides:

```text
normalized spend keys
UNION DISTINCT
proven attributed-revenue keys
```

For the examples:

| Date | Account | Platform | `ext_ad_id` | Why the key exists |
| --- | --- | --- | --- | --- |
| Aug 26 | Acme | Meta | `123` | spend and proven revenue |
| Aug 26 | Acme | Meta | `456` | spend |
| Aug 29 | Acme | Meta | `123` | proven delayed revenue, no spend |

The Aug 29 key comes directly from proven revenue. No fake zero-spend name row is
needed.

### Step 4: join spend and revenue on the stable key

Both sides are already unique at their intended grain:

```text
date + account_id + platform + ext_ad_id
```

The join is therefore ordinary fact joining:

| Date | Account | Ad ID | Spend | Proven sales | Proven commission |
| --- | --- | --- | ---: | ---: | ---: |
| Aug 26 | Acme | `123` | $50 | $50 | $5 |
| Aug 26 | Acme | `456` | $10 | $0 | $0 |
| Aug 29 | Acme | `123` | $0 | $30 | $3 |

There is no name-or-ID `OR`, no token fan-out, no click-share allocation, and no
second weight to collapse physical aliases.

The two exception events remain in reconciliation:

| Event | Publisher/account context | Sales | Commission | Boost ad performance |
| --- | --- | ---: | ---: | --- |
| B: ambiguous `summer-sale` | Acme | $100 | $10 | excluded until one ad is proven |
| D: `hyphenpopsugar-20` | Popsugar | $100 | $10 | excluded; no individual ad proof |

## Side-by-side comparison

| Question | Old method | Proposed ID-first method |
| --- | --- | --- |
| What is the spend grain? | Physical spend/name-shaped rows | One row per date/account/platform/ad ID |
| Where is the ad decision made? | During the spend/revenue join | Once in Layer B before the join |
| Main join key | Date + account/site + name-or-ID token | Date + numeric account ID + platform + stable ad ID |
| Role of ad name | Display field and join identity | Display metadata and deterministic historical fallback |
| Same ad appears in two spend rows | Split with `ad_row_weight` | Rows normalize into one ad/day fact |
| One name identifies two ads | Split by click-share `allocation_weight` | Explicit ambiguous exception |
| Revenue arrives without same-day spend | Generated zero-spend ownership row | Revenue contributes the stable ad/day key directly |
| Publisher tag identifies account but no ad | Retained account/day; absent from ad-level result | Retained in reconciliation; excluded from proven ad performance |
| Can display-name changes alter fact grain? | Yes | No |
| Can attributed money be duplicated by join fan-out? | Prevented by multiple compensating weights | Prevented structurally by unique stable grains |

## What simplicity is gained

The old flow must solve two fan-out problems after joining:

```text
revenue token
    |
    | allocation_weight
    v
candidate ad IDs
    |
    | ad_row_weight
    v
physical output rows
```

The new flow removes both fan-outs before Layer C:

```text
revenue evidence ──proof resolver──> zero or one stable ad ID

raw spend rows ──stable aggregation──> one ad/day row

                 stable-key join
```

The tradeoff is deliberate: the new report can show less ad-attributed revenue
when the old method estimated a match from an ambiguous name. That difference
cannot be silent. Before cutover, every old-attributed/new-unproved amount must
be enumerated, classified, and explicitly approved; every unexplained
difference must be zero.

## One-sentence takeaway

The old method uses spend rows as both facts and an ad-name/ID lookup and then
needs weights to repair the resulting fan-out; the new method proves one stable
ad ID first, normalizes spend to the same stable grain, and joins the two once.