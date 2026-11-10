# Revenue attributed downstream and all brand ad commission

Editable diagram: Revenue attributed downstream and all brand ad commission.excalidraw. SVG is a portable vector preview. The JSON inventory lists source references, including definitions and shared-ledger writers, not only consumers.

The graph is repository-defined, examined November 10, 2026. Project-wide BigQuery INFORMATION_SCHEMA access was denied; external ad hoc views and BI tools are outside the verified scope. Materialized-table lineage is drawn from rebuild procedures and generated serving-view lineage from devops/data/boost-performance.ts. Application boxes group route/query consumers; they are not SQL views.

## Policy conclusion

Charging performance commission on eligible brand-funded ads without inventing a house creator is a viable simpler identity model. revenue_attributed can remain factual. Pricing must remove the null-creator zero-rate rule only for the approved scope and effective dates. Public product rates and eligible website backup can apply when no actual creator exists; private rates still belong to real creators. Preserve product ownership exclusions, non-billable basket items, dated refunds and publisher self-funded exemptions.

It is not enough to change boost_priced_deltas: boost-billing-service currently writes commission only when partnership_creator_account_id is present, and boost-payout-contract converts partnership charge into creator payout basis. Add an explicit Wayward-retained fee classification and reconciliation identity rather than inventing a creator, using UNRESOLVED, or hiding a liability. Decide whether retained fee is a distinct ledger source or an explicit beneficiary kind. Total brand charges must equal creator liabilities plus Wayward-retained amounts and other fees, with rounding controlled.

Brand performance APIs need explicit performance-fee and usage-fee fields or a companion breakdown endpoint. Use shared pricing snapshots for current estimates and frozen ledger for accrued charges, avoiding overlap. Show billable base, rate distribution, exclusions and as-of/status. The current creator-partnership query already reads the pricing snapshot for rates; an earlier discussion describing it as entirely independent was too broad. It also reads actual creator evidence and should continue to do so.

## Proposed validation

Compare actual partnerships, ordinary brand-funded ads and publisher self-funded ads. Check 0% public rates, private overrides, eligible website backup, out-of-brand products, missing identity, event-date refunds, mixed-rate API totals, ledger projections, retained-fee reconciliation and external payouts. Preserve Meta payload, creative access, approvals, comment identity and generation voice. New fees require an explicit scope, activation date and customer disclosure; this artifact does not activate billing.
