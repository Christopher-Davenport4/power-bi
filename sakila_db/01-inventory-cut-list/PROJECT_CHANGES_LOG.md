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

Superseded by entries 13 through 16. Top N was replaced with RANKX measures, and the
tiebreaker was implemented using `film_id` rather than the sort keys described here.

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

---

## 13. Ranking implementation: Top N filter to RANKX measures
**Change:** Replaced Power BI's Top N visual filter with `RANKX` measures that pin
each store internally, `Rank_Store_1` and `Rank_Store_2`.
**Why:** Top N applies to a visual's entire filter context, so with both stores
present it ranks across the combined set rather than producing a separate ranking per
store. Comparing a title's standing at store 1 against its standing at store 2
requires both ranks on the same row, which a visual-level filter cannot express. The
measures use `CALCULATE([Revenue_per_Copy_Held], inventory[store_id] = n)` so each
rank is computed against a fixed store regardless of what filters the visual carries.

The ranking population for each store is restricted to titles that store actually
stocks. An unstocked title returns blank for revenue per copy held, blank coerces to
zero in a numeric comparison, and in ascending order those zeros would occupy the
worst positions and displace every genuinely weak title.

---

## 14. Ranking method: Dense to Skip
**Change:** Changed every `RANKX` call from `Dense` to `Skip`.
**Why:** Dense ranking assigns tied values the same rank and continues the sequence
without a gap, so rank values are not positions. Filtering to rank 200 or below was
therefore returning more than 200 titles at each store, and every threshold in the
analysis was measuring something other than what it stated. The error was visible as
a mismatch between two figures that should have agreed: the count of titles
qualifying as bottom 200 at both stores returned 139, while the ranking over that
same group topped out at 132, meaning seven pairs had been collapsed into shared rank
values.

Under Skip, tied values share a rank and the following rank advances past them, so
rank 200 or below corresponds to roughly the worst 200 positions. The count of titles
in both stores' bottom 200 fell from 139 to 111 after the change. The 139 was an
artifact of inflated per-store lists.

---

## 15. Deterministic ordering: film_id as final tiebreaker
**Change:** Rank expressions sort on `[Revenue_per_Copy_Held] * 1000000 +
MAX(film[film_id])` rather than on revenue per copy held alone. Ranking populations
were widened to `ALL(film[title], film[film_id])` so the identifier is available
inside the iteration.
**Why:** Revenue per copy held produces ties readily, since different combinations of
revenue and copy count resolve to the same rate. Without a tiebreaker, two titles with
identical performance can land on either side of the cutoff for no stated reason, and
a cut list has to be defensible title by title. Multiplying the rate by a constant
larger than the identifier range keeps revenue as the dominant sort key while
guaranteeing that `film_id`, which is unique, settles anything that would otherwise
tie.

---

## 16. Cut criterion: pooled revenue rejected for worse-of-two-store-ranks
**Change:** Titles are ordered for the cut list by their worse rank across the two
stores, with the better rank used only to break ties. Ranking on chain-wide revenue
per copy held was implemented first and rejected.
**Why:** Pooling revenue across both stores produces a copy-weighted average that can
mask a title performing strongly at one location. A title severely weak at one store
and healthy at the other can post a poor pooled rate and be cut, which is the failure
this analysis exists to prevent. Ordering by the worse of the two store ranks means a
title only advances toward the cut list if it is weak at its stronger store as well.

This is equivalent to widening the per-store cutoff symmetrically until the
intersection of the two bottom lists reaches the target count. A title enters that
intersection at whatever its worse rank is, so ranking on the worse rank produces the
same ordering in a single pass.

Confirmed empirically: all 111 titles in both stores' bottom 200 fall inside the top
200 of the worse-rank ordering. The two criteria nest correctly, which was not true
of the pooled-revenue version.

---

## 17. Finding: 200 titles cannot be justified on chain-wide weakness alone
**Change:** The cut list is built in two tiers rather than as a single undifferentiated
200.
**Why:** Only 111 titles fall in both stores' bottom 200. The remaining 89 required
widening the cutoff beyond that threshold, and each is weaker at one store than the
other. Presenting all 200 as equivalent would overstate the confidence behind the
second group. The two tiers are reported separately so the difference in evidence is
visible.

---

## 18. Worse-of-two-ranks rejected as the fill criterion
**Change:** Ordering the full 200 by worse-of-two-store-ranks was implemented and then
abandoned. Entry 16 describes the criterion; it is retained here as a record of why it
did not survive contact with the data.
**Why:** A title needs to be weak at both stores to receive a low worse-rank, and few
titles are. To accumulate 200 the threshold had to climb until it was admitting middle
performers. The case that surfaced it: a title ranked 336 at store 1 and 102 at store
2 was landing on the cut list. With 759 titles stocked at store 1, rank 336 is near the
45th percentile, which is not a weak title. The criterion was cutting exactly what it
was designed to protect.

The underlying problem is that the criterion and the target were incompatible. Under a
strict "weak at both stores" standard only 111 titles qualify. Anything past that is
reached by relaxing the standard, not by finding more weak titles.

---

## 19. Percentile guard tested and set aside
**Change:** Tested a guard requiring any filled title to fall below a percentile
threshold at the other store. Bottom third was tested first, then median. Neither was
adopted.
**Why:** The guard was intended to allow filling from titles weak at one store while
preventing removal of anything performing acceptably at the other. Store 1 stocks 759
titles and store 2 stocks 762, so bottom third is rank 253 and 254 respectively, and
median is 380 and 381.

Bottom third yielded 22 eligible titles against 89 needed. Median yielded 55. Neither
fills the target, and both cut titles that perform acceptably at one location, which
is the outcome the analysis exists to avoid. Median was the more permissive option and
still fell short, so relaxing further would have meant cutting clearly adequate titles
to hit a number.

Testing the stricter threshold first was deliberate. Starting at median and stopping
there would have produced a defensible-looking list without revealing that a stricter
standard was available.

---

## 20. Fill criterion: titles stocked at a single store
**Change:** The remaining 89 titles are drawn from titles stocked at only one store
that fall in that store's bottom 200, ranked by revenue per copy held and cut worst
first. 106 titles qualify, so 89 are taken and 17 are retained.
**Why:** A title stocked at one store cannot be removed from a location where it earns,
because there is no second location. The risk that drove the entire analysis does not
apply to this group. Removing one takes a weak title out of the only place it exists.

This is stronger evidence than the percentile guard produced. The 55 titles the median
guard would have allowed were being cut despite performing acceptably somewhere. None
of the 89 are.

479 of the 1,000 titles are stocked at a single store, so the eligible pool was large
enough to fill the target without relaxing any standard.

---

## 21. Final cut list: two tiers, no compromised titles
**Change:** The list is 200 titles in two tiers. Tier one is 111 titles in the bottom
200 at both stores. Tier two is 89 titles stocked at a single store and in that store's
bottom 200.
**Why:** Every title on the list is weak in every location that carries it. Tier one
titles are weak at both stores. Tier two titles are weak at their only store. No title
is removed from a location where it performs acceptably, which was the failure the
analysis was built to prevent.

The tiers are reported separately because the evidence behind them differs. Tier one
rests on agreement between two independent rankings. Tier two rests on a single
store's ranking, evaluated against a population that includes titles stocked at both
stores.

---

## 22. Distribution check: no natural break at the cut line
**Change:** Plotted revenue per copy held across every title each store stocks, sorted
ascending, to test whether the bottom of the catalog separates from the rest.
**Why:** If performance dropped off sharply at some point, that break would be the
defensible place to cut and the target of 200 could be evaluated against it. It does
not. The curve is flat at the low end, rises slowly and steadily through the middle,
and jumps only at the final two titles.

The consequence is that 200 is a business target rather than a boundary the data
identifies. Titles immediately inside and outside the line are indistinguishable on
performance, and cutting 180 or 220 would be equally supportable. This is reported in
the recommendation rather than kept in the working analysis, because acting on the
list while assuming the data identified a natural cutoff would be acting on a belief
the analysis contradicts.

---

## 23. Thin data check on the cut list
**Change:** Checked rental volume behind every title on the cut list, per store rather
than in aggregate.
**Why:** Revenue per copy held is a rate, and a rate computed from two or three
transactions carries no information about performance. The aggregate rental count was
checked first and showed a minimum of five, but that figure combines both stores and
could conceal a title with four rentals at one store and one at the other.

Recomputed per store, one title on the list has four rentals at store 1 against nine at
store 2. Every other title clears five at each store that stocks it. The one title
remains on the list, since its store 2 volume supports the same conclusion. No
exclusions were made.

---

## 24. Join integrity confirmed
**Change:** Verified that every rental has exactly one payment before treating revenue
figures as final.
**Why:** Revenue reaches film through `payment` to `rental` to `inventory`. If any
rental carried multiple payments, the join would multiply rows and inflate revenue for
the affected titles. Equal row counts in `payment` and `rental` were consistent with a
one-to-one relationship but did not establish it. Counting rentals with at least one
matching payment returned 16,044, matching both table counts exactly.

---

## 25. Store totals: near-identical revenue, different catalogs
**Change:** Recorded total revenue by store: 33,679.79 at store 1 and 33,726.77 at
store 2, a difference of 0.14 percent.
**Why:** This was an open question from the start of the analysis. If one store were
substantially weaker, a chain-wide cut list built on pooled totals would be dominated
by the stronger store's volume, and that imbalance would be the headline. It is not.
The two stores earn effectively the same revenue while stocking meaningfully different
catalogs, with 479 of 1,000 titles carried at only one location. The divergence between
the stores is in what they stock, not in what they earn.

---

## 26. Cut list delivered as a spreadsheet, not a visual
**Change:** The 200 titles are exported to CSV. The report carries the reasoning behind
the list, not the list itself.
**Why:** A 200 row list is a working document. It gets filtered, sorted, and worked
through against physical inventory, none of which a chart supports. Rendering it as a
visual would make it harder to use without making it easier to understand. The report
covers what cannot be read off the spreadsheet: why the analysis window differs from
the one requested, what the two tiers mean, and that the cut line is a target rather
than a boundary in the data.

---

## 27. Spreadsheet columns reported per store rather than aggregated
**Change:** The exported list carries revenue per copy held and copies held as separate
columns for each store rather than as combined figures.
**Why:** The ranking was computed per store, so an aggregate revenue figure is a
copy-weighted average that appears nowhere in the logic that put a title on the list.
Copies held per store also identifies where shelf space is recovered, which the
combined number hides. A blank in either store's column indicates the title is not
stocked there, which carries the presence information without a separate column.
Aggregates can be derived from the per-store figures; the reverse is not true.
