# Sakila Data Storytelling

An independent, from-scratch project focused on two skills: dimensional data
modeling and telling clear stories with data. I use the Sakila sample database
as a consistent foundation and build a series of dashboards, each starting from
a business question and an audience rather than from a chart type.

Unlike guided tutorial work, every design decision here is my own and documented
as such.

## Approach

Each scenario follows the same order: why before what before how.

1. **Why** — who the audience is and what they should do differently after seeing it.
2. **What** — the question of the data and the comparison it implies
   (trend, ranking, part-to-whole, relationship, distribution).
3. **How** — the model, the Power Query shaping, the DAX, and last of all the
   chart. The visual is an output of the message, not a starting point.

## Structure

- `00-model/` — the shared foundation: Sakila extracted and shaped in Power
  Query, modeled as a star schema, with base DAX measures validated against raw SQL.
- `NN-<scenario>/` — one folder per story scenario, each with its dashboard and
  a short write-up of the question, audience, and chart rationale.

## Documentation

- `MODELING_LOG.md` — grain decisions, dimension design, and the reasoning behind
  each modeling choice.
- Chart-decision log (in the root README, added as scenarios are built) — for each
  visual: the one-sentence takeaway, the chart chosen, and what was rejected and why.

## Tools

Power BI, Power Query (M), DAX, MySQL (Sakila). Data modeling with a star schema.
