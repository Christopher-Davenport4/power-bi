# Sakila Inventory Performance Analysis

A Power BI analysis of title-level rental performance across a two-store DVD rental
chain, built to support an inventory reduction decision. The work covers data
modeling from a normalized source schema, measure design, per-store performance
ranking, and a report structured around a single recommendation.

---

## A note on scope

This is a data modeling and reporting project. The focus is on turning a normalized
OLTP schema into a model that answers a specific business question, designing
measures that map to the decision being made, and presenting the result so it can
be acted on without narration.

The business situation is constructed. Sakila is a sample database with no real
operators, so the role and the decision described here are written to fit what a
two-store rental chain would plausibly face. The data, the measures, and the
findings are real properties of the database.

## A note on authorship

I built the data model, wrote the DAX, and made the design decisions in this
project. I used Claude to talk through modeling approaches, pressure test the
metric choice, and draft documentation. The reasoning behind the major decisions is
recorded in `PROJECT_CHANGES_LOG.md`.

---

## Business question

A regional manager has to remove 200 titles from inventory. She needs to know which
titles, and whether one list applies across the chain or whether each store needs
its own.

The second half is the part that carries risk. The two stores do not hold identical
inventory, so a single list built on chain-wide totals can remove titles that earn
at one location in order to hit a number set centrally. Confirming whether that
would happen is a precondition for recommending anything.

---

## Data

Sakila models a DVD rental chain with two stores, a shared film catalog, per-store
inventory, and customer rental and payment transactions.

| Table | Rows |
|---|---|
| film | 1,000 |
| inventory | 4,581 |
| payment | 16,044 |
| rental | 16,044 |

Rental activity spans 2005-05-24 to 2006-02-14. Inventory is held per store, 2,270
copies at store 1 and 2,311 at store 2. A title may be stocked at one store or both.

Revenue flows from `payment` through `rental` to `inventory` to `film`. Store
assignment comes from `inventory.store_id`, since that is where a copy physically
sits, rather than from customer registration.

---

## Performance measure

Titles are ranked on revenue per copy held: collected revenue for a title at a
store, divided by the number of copies that store holds.

```
Revenue = SUM(payment[amount])
Copies Held = COUNTROWS(inventory)
Revenue per Copy Held = DIVIDE([Revenue], [Copies Held])
```

Total revenue was rejected as the ranking metric. Titles are stocked at an average
of 4.6 copies, so total revenue rewards stocking depth rather than performance. The
cost a removal recovers is the shelf slot, which makes per-copy return the measure
that maps to the decision.

Sakila holds `rental_rate` and `replacement_cost` on `film` and collected revenue in
`payment.amount`, but nothing for acquisition cost. Every figure here is revenue or
utilization. No claim in this project is a profitability claim.

---

## Report structure

The report is a single file with two pages.

- **Recommendation.** The cut list and the evidence behind it. Built to be read
  without narration, so it carries the detail needed to answer the obvious
  questions on its own.
- **Exploration.** The working analysis: the checks run, the distribution of
  performance across the catalog, and the alternatives ruled out.

The two are kept separate so the working analysis does not leak into the
communication.

---

## Known limitations

- **Uneven activity.** Rental volume is concentrated between May and August 2005,
  with a single day of activity in February 2006. Ranking is done on total
  performance across the full range rather than a calendar quarter, since the
  question is how much each title earned per copy, not when it earned it.
- **Thin data at the title level.** A title held in one copy with a single rental
  produces a per-copy figure off almost no observations. Copies held is reported
  alongside the ranking so these cases stay visible.
- **Unmatched inventory.** Not every title is stocked at both stores, so per-store
  rankings are drawn from different pools. Each store is ranked only over what it
  holds, and cross-store comparison is done separately on the titles both stores
  carry.

---

## Documentation

- `PROJECT_CHANGES_LOG.md`: a running record of the decisions made and the
  reasoning behind each.
- `01-inventory-cut-list/01_context.md`: the audience, action, mechanism, and tone
  work completed before any analysis, and what the data can and cannot support.
