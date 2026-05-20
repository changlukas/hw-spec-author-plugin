Findings:

1. **D** landed faithfully: removed from P0, moved to P1, and its rework explicitly targets the verified `examples/wctmr/doc/registers.md:104` false-positive by requiring an external vendor/standard attribution token and regression-zero hits on internal `§` cross-references.

2. **K** landed faithfully: defaults to `WARN`, never `FAIL`, and now has a measured false-positive/precision ship gate before merge.

3. **B** landed faithfully: revised to TP ID uniqueness; duplicate IDs `FAIL`, numbering gaps `INFO`.

4. **M** landed faithfully: register bit-overlap / width-overflow lint added to P0 and included in Stage 1.

5. **N/O** landed faithfully: cross-file scalar consistency and stale WAIVERS anchor checks added as P1 items.

6. **Nine open questions** landed faithfully: section 7 resolves Q-E1, Q-F1, Q-F2, Q-G1, Q-H1, Q-H2, Q-K1, Q-L1, Q-L2 per aggregate consensus/adjudication.

Verdict: **Yes, the revision faithfully applies the aggregate’s section 5 recommendations.**

New inconsistency found: **minor**. Item H says `D1.delta_from_plan` must never be freeform and requires a written `PLAN.md` or `/spec-init` transcript, otherwise N/A is stated loudly. But its worked-example acceptance still allows adding a one-line `"no delta detected"` annotation, which can read like the freeform declaration Q-H1 explicitly rejected unless tied to a written plan artifact. This should be tightened to “N/A declaration unless a written plan artifact exists.”
