# Sakila: Data Modeling and Visual Storytelling

A Power BI project built on the Sakila sample database, which models a DVD rental
chain with stores, staff, inventory, customers, and rental transactions.

I am using this project to work on two things: dimensional data modeling, since
Sakila arrives as a normalized OLTP schema and turning it into a star schema is the
exercise, and communicating findings clearly, which I am working on alongside Cole
Nussbaumer Knaflic's *Storytelling with Data*.

Each scenario starts with a business question and an audience. The chart comes last
and follows from the message rather than the other way around.

## Structure

- `PROGRESS_LOG.md` — running record of decisions, findings, and open items.
- `01-inventory-cut-list/` — first scenario. Context document, report, and
  supporting queries.

Each scenario folder holds a context document written before any analysis, the
Power BI report, and a short write-up of what the analysis found.

Reports are built with two pages. The recommendation page carries the finding and
is the only thing the stakeholder would see. The exploration page holds the working
analysis, the checks I ran, and the alternatives I ruled out.

## Scenarios

**01. Inventory cut list.** A regional manager has to cut 200 titles from inventory
this quarter. The question is which titles, and whether one chain-wide list works
or whether each store needs its own. A single list built mostly from the stronger
store's volume could gut the other store's assortment, so the analysis checks for
that before making a recommendation.

## Notes on the data

The business situation in each scenario is fictional. Sakila is a sample database
with no real stakeholders, so the audiences and decisions are constructed to fit
what that business would plausibly face.

Sakila has no acquisition cost or margin data, so everything here is measured in
revenue or utilization rather than profit.

## Tools

Power BI, Power Query (M), DAX, MySQL.
