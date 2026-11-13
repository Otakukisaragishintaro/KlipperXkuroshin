# KlipperXkuroshin

[English](README.md) | [简体中文](README.zh-CN.md)

An independent patch proposal for Klipper's `bed_mesh` rapid-scan workflow.
The intended upstream target is [Klipper3d/klipper](https://github.com/Klipper3d/klipper).

## Current proposal

This repository contains a review candidate that adds configurable repeated
rapid scans and per-path-point median absolute deviation (MAD) filtering.

- The scan path is materialized once and replayed for every pass.
- Results are aligned by their path-point index.
- The filter operates on `ProbeResult.bed_z` values only.
- Accepted heights are averaged; one result per path point is passed to the
  existing bed-mesh finalization callback.
- Eddy's per-window frequency averaging and frequency-to-height conversion
  are not modified.
- Existing single-pass behavior remains the default.

## Configuration proposal

```ini
[bed_mesh]
rapid_scan_repeats: 5
rapid_scan_mad_multiplier: 3.0
rapid_scan_outlier_min: 0.020
```

The default repeat count is 1. Values of 3 or greater are supported, including
even values; 2 is rejected. The 0.020 mm absolute floor is provisional and has
not been calibrated against a representative printer population.

For each path point:

```
center = median(z)
MAD = median(abs(z - center))
threshold = max(outlier_min, multiplier * 1.4826 * MAD)
```

Samples farther than the threshold from the median are rejected. At least two
samples must remain, after which their arithmetic mean is used.

## Review notes

The source review confirmed that Python executes the `finally` block before
Klipper's `_process_commands()` dispatches `gcode:command_error`. Eddy's
registered error handler checks whether `_gather` is non-`None`, so the normal
cleanup path does not call `end_probe_session()` twice. The earlier concern
about inevitable double cleanup was incorrect. The patch now documents this
ordering and rejects an impossible empty rapid-scan path before starting a
probe session.

See [CHANGELOG.md](CHANGELOG.md) for the review and hardening record.

## Patch baseline and status

- Upstream: `Klipper3d/klipper`
- Base commit: `29d4ec2d6d840ee839333306003a400da186d41d`
- Patch: `patches/0001-bed-mesh-rapid-scan-mad-filter.patch`
- Status: experimental review candidate; not yet validated on hardware or
  accepted by upstream.

See [docs/testing.md](docs/testing.md) before testing. This project does not
claim upstream approval or compatibility with every Klipper revision.
