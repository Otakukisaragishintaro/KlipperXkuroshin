# Changelog

## 2026-11-14 — Source review and defensive hardening

### Verified against upstream source

- Reviewed `klippy/gcode.py` at base commit
  `29d4ec2d6d840ee839333306003a400da186d41d`.
- Confirmed `_process_commands()` catches a G-code command error only after
  the command handler has unwound. Python therefore executes the rapid-scan
  handler's `finally` block before Klipper sends `gcode:command_error`.
- Confirmed Eddy's `_handle_command_error()` calls
  `end_probe_session()` only when `_gather` is not `None`. Since the
  `finally` block has already ended the session and cleared `_gather`, the
  previously reported inevitable double-cleanup issue does not occur on this
  path. The earlier concern was a review error, not a confirmed code defect.
- Reviewed `EddyGatherSamples`: probe requests are queued and analysis
  results are appended in request order, supporting path-index grouping when
  every pass queues the same path in the same order.
- Reviewed `ProbeManager.iter_rapid_path()`: the generated path includes
  probe points, non-probing travel, faulty-region substitutions, optional
  overshoot travel, and the optional zero-reference probe point. Materializing
  the generator once preserves that same sequence for each pass.

### Change made

- Added a guard that rejects a generated rapid-scan path containing no probe
  points before a probe session is started.
- Added an explanatory comment documenting the verified command-error /
  `finally` ordering.
- Expanded the validation checklist to cover both behaviors.

### Not yet verified

- The patch has not yet been applied to a local Klipper checkout with
  `git apply --check`.
- Python compilation, automated tests, hardware tests, and noise-floor
  calibration have not yet been run.
- The proposed `rapid_scan_outlier_min: 0.020` remains provisional.
