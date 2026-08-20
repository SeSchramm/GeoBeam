# Changelog

All notable changes to GeoBeam are documented here.

## [0.2.0] — 2026-08

### Fixed

**`beam_f3_coordinates.csv` was overwritten with the Beam_F3_irl result (affects v0.1.0 and v0.1.1).**

`calculate_beam_f3_coordinates()` wrote `measurements/beam_f3_coordinates.csv`
itself, using hard-coded row labels `Beam_F3` and `Ref_Beam_F3`. Because `main()`
calls that function twice — once with `x_f3_geo`/`y_f3_geo` and again with
`x_f3_irl`/`y_f3_irl` — the second call silently overwrote the file produced by
the first. The result was a file containing the **in-real-life (irl) coordinates
labelled as the geodesic (geo) coordinates**, with nothing to indicate the
substitution.

*Who is affected.* Only runs in which **both `x_f3_irl` and `y_f3_irl` were
supplied**. Runs using `x_f3_geo`/`y_f3_geo` alone call the function once and are
unaffected. Note that the batch example in the README passed `x_f3_irl` and
`y_f3_irl`, so users following that example are affected.

*How to check existing output.* `f3_coordinates.csv` is written before the irl
calculation runs and therefore still holds the correct geo coordinates. For an
affected subject, `f3_coordinates.csv` and `beam_f3_coordinates.csv` will
disagree. Comparing file modification times also works: an affected
`beam_f3_coordinates.csv` is newer than or equal in age to
`beam_f3_irl_coordinates.csv`.

*Unaffected outputs.* `f3_all_coordinates.csv` and `ref_points_coordinates.csv`
are assembled from in-memory variables rather than re-read from disk, and are
correct. `beam_f3_irl_coordinates.csv` and `f3_coordinates.csv` are also correct.
If your analysis drew Beam_F3 coordinates from `f3_all_coordinates.csv`, your
results are unaffected.

*Fix.* All CSV export is now performed by the caller, which knows which variant
it is holding. `calculate_beam_f3_coordinates()` no longer writes files.

### Changed

- `run_all()` now requires an explicit `base_path` argument instead of a
  hard-coded path inside the function body, and returns `(processed, failed)`.
  It prints a summary listing every failed subject, so that a subject failing
  partway through a long batch is not lost in the scrollback.
- The Beam_F3 path visualisation in `main()` is now optional and off by default,
  controlled by `visualize_beam_f3_plot`. Previously it opened an interactive
  window on every subject, blocking batch processing until closed manually.
  `visualize_beam_f3()` gained `off_screen`, `output_dir` and `screenshot_name`
  parameters; with `beam_f3_off_screen=True` (the default) it writes a PNG.
- `calculate_beam_f3()` combined with `calculate_vertex_Real=False` now raises a
  clear `ValueError` rather than silently recomputing Vertex_Real internally.
  Beam_F3 is defined relative to Vertex_Real, so the combination was never valid.

### Performance

These changes do not alter any computed coordinate; they remove redundant work.

- `calculate_beam_f3_coordinates()`, `find_ref_beam_f3()`,
  `find_ref_point_on_nz_vertexreal()` and `determine_electrodes_geo()` now accept
  optional `geoalg`, `Cz_AP` and `vertex_Real` arguments, and `main()` passes the
  values it has already computed. Previously each of these functions recomputed
  them from scratch. Per subject this reduces `find_Cz_AP()` from five
  evaluations to one and `find_vertex_Real()` from six to two, eliminating
  roughly 36 redundant exact-geodesic queries.
- `PyGeodesicAlgorithmExact` is now constructed once per subject rather than four
  times.
- `validate_faces()` is vectorised instead of looping over triangles in Python
  (~80x faster on an undecimated scalp mesh). Output is identical, verified
  against the previous implementation on 500 randomised cases including
  degenerate triangles.

## [0.1.1] — 2026-06

State of the code released alongside the preprint. Functionally identical to
0.1.0 plus documentation revisions. **Contains the `beam_f3_coordinates.csv`
defect described under 0.2.0.**

## [0.1.0] — 2026-06

Initial release.
