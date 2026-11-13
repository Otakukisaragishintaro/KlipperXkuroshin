# Validation plan

This checklist is intentionally explicit: a patch file existing in this
repository does not mean it has passed these tests.

## Patch and static checks

1. Check out the exact upstream base commit listed in the repository README.
2. Apply the patch with `git apply --check`, then apply it.
3. Run Python syntax compilation for `klippy/extras/bed_mesh.py`.
4. Run the upstream bed-mesh regression tests.
5. Confirm the default repeat count of 1 preserves single-pass behavior.
6. Confirm a repeat count of 2 is rejected and counts 3, 4, and 5 are accepted.
7. Confirm result-count mismatches and fewer than two retained samples produce
   a clear command error.
8. Confirm the probe session is ended if scanning, result retrieval, filtering,
   or finalization raises an error.

## Behavioral checks

- Compare a single-pass scan with unmodified Klipper.
- Verify every pass replays exactly the same rapid path, including overshoot
  travel and probe points.
- Verify the path-point order and result count remain consistent.
- Verify zero-reference points and faulty-region replacement points are
  retained and filtered like other path points.
- Verify axis-twist compensation is applied before MAD aggregation through
  the existing `probe:update_results` event.
- Verify one aggregated result per path point reaches the existing finalizer.
- Verify the filter never modifies Eddy's per-window frequency averaging or
  frequency-to-height calibration.

## Controlled outlier checks

Use synthetic or controlled measurements to cover:

- Stable values with no outliers.
- One isolated high or low outlier.
- Even and odd repeat counts.
- MAD equal to zero, protected by the absolute threshold floor.
- A case where fewer than two values remain (must fail safely).
- A result list whose length does not equal
  `rapid_scan_repeats * probe_point_count` (must fail safely).

## Noise-floor evaluation

Collect repeated measurements at a fixed location under stable thermal and
mechanical conditions. Evaluate the absolute floor against observed noise and
measurement resolution. Do not treat 0.020 mm as a universally validated
threshold.

## Upstream readiness

Before opening an upstream pull request, update the official configuration
reference and add appropriate automated tests. Run the relevant Klipper test
suite and record the exact results. The submitter must provide their own
accurate Signed-off-by identity in accordance with upstream contribution
requirements.
