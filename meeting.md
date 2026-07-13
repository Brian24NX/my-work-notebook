Planned change: make the distance calculation more robust to noisy data points

What we do today. For each camera, we learn a simple rule — "where an animal's feet appear in the frame → how far away it is (in meters)" — by fitting a straight line through a set of reference measurements. We currently fit that line with standard least-squares, which can get pulled off-course by a few bad reference points (a mis-detected foot, an odd measurement).

The change. Switch the line-fitting to a robust method (Theil-Sen) that shrugs off those few bad points — think median instead of average: a couple of wild values no longer skew the result. It's a one-line change to the fitting step; we then re-run the calibration and regenerate the distances.

Why we're doing it. Nick suggested looking at noise-robust methods for the distance calc, so we tested it head-to-head:
- Robust fitting (Theil-Sen) tightened distance accuracy by ~5% (0.95 m → 0.91 m), by not getting fooled by outlier points.
- We also tried another robust method (RANSAC) — it was worse on our small per-camera datasets, so we're not using it.

A useful side-finding. The handful of "problem" cameras don't have a noise problem — they have a placement problem (aimed too low/flat, so foot position barely reflects distance). No fitting method fixes that; those would need re-aiming in the field.

Scope & risk. Very small, low-risk: a one-line method swap plus a re-run. It only tightens the numbers — it doesn't change the pipeline's structure or outputs' format. Distance is already merged, so this goes in as a small, self-contained follow-up.








Planned change: make the distance calculation more robust to noisy data points

What we do today. For each camera, we learn a simple rule — "where an animal's feet appear in the frame → how far away it is (in meters)" — by fitting a straight line through a set of reference measurements. We currently fit that line with standard least-squares, which can get pulled off-course by a few bad reference points (a mis-detected foot, an odd measurement).

The change. Switch the line-fitting to a robust method (Theil-Sen) that shrugs off those few bad points — think median instead of average: a couple of wild values no longer skew the result. It's a one-line change to the fitting step; we then re-run the calibration and regenerate the distances.

Why we're doing it. Nick suggested looking at noise-robust methods for the distance calc, so we tested it head-to-head:
- Robust fitting (Theil-Sen) tightened distance accuracy by ~5% (0.95 m → 0.91 m), by not getting fooled by outlier points.
- We also tried another robust method (RANSAC) — it was worse on our small per-camera datasets, so we're not using it.

A useful side-finding. The handful of "problem" cameras don't have a noise problem — they have a placement problem (aimed too low/flat, so foot position barely reflects distance). No fitting method fixes that; those would need re-aiming in the field.

Scope & risk. Very small, low-risk: a one-line method swap plus a re-run. It only tightens the numbers — it doesn't change the pipeline's structure or outputs' format. Distance is already merged, so this goes in as a small, self-contained follow-up.
