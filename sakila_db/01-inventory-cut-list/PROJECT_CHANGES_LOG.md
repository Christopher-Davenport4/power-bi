# Project Changes Log

A record of the decisions made in this project and the reasoning behind each, kept
so the choices can be referenced later without reconstructing them from memory.

---

## 1. Connection: DirectQuery to Import mode
**Change:** Loaded all Sakila tables into Power BI using Import mode rather than
DirectQuery.
**Why:** Power BI does not natively support DirectQuery against MySQL, and the
MariaDB ODBC workaround introduced version-specific problems on a previous project.
Sakila is small enough that an imported model refreshes in seconds, so DirectQuery
buys nothing here and costs connector maintenance.

---

## 2. Ranking metric: total revenue to revenue per copy held
**Change:** Titles are ranked on revenue per copy held rather than total revenue.
**Why:** Titles are stocked at an average of 4.6 copies, and stocking depth is not
evenly distributed. Total revenue therefore rewards titles that were stocked heavily
rather than titles that earn. The cost an inventory removal recovers is the shelf
slot, so the measure has to express return per slot. A title earning $20 across four
copies is a weaker use of shelf space than one earning $15 across two.

---

## 3. Profitability excluded from scope
**Change:** All performance claims are stated in revenue or utilization terms. No
margin or profitability figure appears in the analysis.
**Why:** Sakila carries `rental_rate` and `replacement_cost` on `film` and collected
revenue in `payment.amount`, but nothing for what a copy cost to acquire. Margin
cannot be derived from what is present, so stating any figure as profit would be
unsupported.

---

## 4. Revenue source: rental_rate to payment.amount
**Change:** Revenue is calculated from `payment.amount` rather than from
`film.rental_rate`.
**Why:** `rental_rate` is the list price on the title record, not what was collected.
`payment.amount` reflects actual transactions including any duration adjustments,
which is the figure a revenue-per-slot decision should rest on. Payment and rental
carry identical row counts, which is consistent with one payment per rental and no
inflation on the join.

---

## 5. Store assignment: customer.store_id to inventory.store_id
**Change:** Store attribution comes from `inventory.store_id`, not from the store a
customer is registered to.
**Why:** `customer.store_id` records where a customer signed up. `inventory.store_id`
records where the physical copy sits. The two disagree frequently, and the decision
here is about which shelf a copy occupies, so the inventory path is the correct one.
This is the same join-path distinction identified in earlier work against this
schema.

---

## 6. Analysis window: single quarter to full date range
**Change:** Considered restricting the analysis to the busiest quarter, then decided
to rank titles across the entire date range.
**Why:** Rental volume ramps through May and June 2005 and peaks in July and August.
Scoping to the peak quarter would have measured performance during summer only, so a
title that rented steadily in spring and slowed in summer would be ranked as an
underperformer on a seasonal artifact. Ranking across the full range covers both
slower and busier periods, and the question is how much each title earned per copy
rather than when it earned it.

The month-level distribution is retained on the exploration page as the basis for
this decision, not as an input to the ranking.

---

## 7. Per-store rankings limited to stocked titles
**Change:** Each store's ranking includes only the titles that store actually holds.
**Why:** Inventory is not matched across stores. A title absent from a store's
ranking because the store does not stock it would otherwise read as a title with no
revenue, which is a different finding entirely. A store cannot remove a title it
does not hold, so including unstocked titles in its ranking would produce
unactionable rows.

This falls out of the model rather than requiring a filter: with a store selected,
copies held evaluates to blank for titles that store does not stock, so the per-copy
measure is blank and the row does not render.

---

## 8. Cross-store comparison run separately on matched titles
**Change:** The two per-store rankings are built first and compared afterward, with
the cross-store comparison restricted to titles both stores carry.
**Why:** The per-store lists are drawn from different pools, so a title's presence in
one list and absence from the other can reflect stocking rather than performance.
Restricting the comparison to matched titles isolates the difference that matters:
whether a title that ranks poorly at one store performs acceptably at the other.
That case is the one that makes a chain-wide removal list the wrong instrument.

---

## 9. Bottom 200 selection and tie handling
**Change:** The bottom 200 is currently selected using Power BI's Top N filter. A
deterministic tiebreaker is required before the list is treated as final.
**Why:** Top N resolves ties arbitrarily. Revenue per copy held produces ties readily,
since different combinations of revenue and copy count land on the same rate, so two
titles with identical performance can split at the boundary with one removed and one
retained for no stated reason. A cut list has to be defensible title by title.
Ordering will be made deterministic by adding secondary sort keys, revenue per copy
ascending, then total revenue ascending, then title.

---

## 10. Report split into recommendation and exploration pages
**Change:** The report is one file with two pages rather than a single combined view.
**Why:** The recommendation page carries the decision and the evidence behind it. The
exploration page holds the working analysis, the checks run, and the alternatives
ruled out. Combining them would put the process in front of the decision, which
inverts what the reader needs. Keeping both in one file means both sit on the same
model and the same measures, so the two cannot drift apart.

---

## 11. Recommendation written for reading, not presentation
**Change:** The report is built to be read without narration rather than presented
live.
**Why:** Detail requirements differ. A presented report can carry less because
questions are answered in the room. A report that circulates on its own has to
anticipate them. Building for the second case and presenting from it would work;
building for the first and circulating it would not.

---

## 12. Reporting order: finding stated before it exists
**Change:** The one-sentence summary of the recommendation is left unwritten until
the analysis supports one.
**Why:** A summary written in advance is a hypothesis, and committing to it early
biases which result gets pursued. The working assumption going in was that 200
titles underperform consistently across the chain. That assumption is recorded so it
can be tested rather than confirmed.
