SIDEKIC — recent progress

The pipeline is now complete end-to-end: detect → count → species → distance → behavior.

Merged to main:
- Distance — all 100 cameras. 0.95 m on the 64 hand-measured cameras, plus a new install-geometry model that covers the 36 un-measured ones with no fieldwork (~1.28 m), QC-validated against the ground truth. (PR #26)
- Productionizing — one push-button program that ran the whole archive: 14,577 clips, end to end. (PR #23)

In review / finishing up:
- Behavior (motion-based line) — validated at macro-F1 0.359 (~2.3× the naive baseline); the precision fixes beat the plain version in a controlled A/B, and I caught + fixed a scoreboard bug along the way. (PR #33, open)
- Completing the deliverable now — writing behavior labels into the distances table so the final output has species + count + distance + reaction + behavior all filled (running on the cluster).

Also: answered Danni's GT-label-budget question (~5–6 labels per new camera → ~1 m), and cleared Danni's + Nick's PR reviews.

Next: finish the deliverable table → get #33 reviewed → a head-to-head / ensemble of the motion model with Noah's skeleton model.







Planned change: make the distance calculation more robust to noisy data points

What we do today. For each camera, we learn a simple rule — "where an animal's feet appear in the frame → how far away it is (in meters)" — by fitting a straight line through a set of reference measurements. We currently fit that line with standard least-squares, which can get pulled off-course by a few bad reference points (a mis-detected foot, an odd measurement).

The change. Switch the line-fitting to a robust method (Theil-Sen) that shrugs off those few bad points — think median instead of average: a couple of wild values no longer skew the result. It's a one-line change to the fitting step; we then re-run the calibration and regenerate the distances.

Why we're doing it. Nick suggested looking at noise-robust methods for the distance calc, so we tested it head-to-head:
- Robust fitting (Theil-Sen) tightened distance accuracy by ~5% (0.95 m → 0.91 m), by not getting fooled by outlier points.
- We also tried another robust method (RANSAC) — it was worse on our small per-camera datasets, so we're not using it.

A useful side-finding. The handful of "problem" cameras don't have a noise problem — they have a placement problem (aimed too low/flat, so foot position barely reflects distance). No fitting method fixes that; those would need re-aiming in the field.

Scope & risk. Very small, low-risk: a one-line method swap plus a re-run. It only tightens the numbers — it doesn't change the pipeline's structure or outputs' format. Distance is already merged, so this goes in as a small, self-contained follow-up.
