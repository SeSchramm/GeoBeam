# GeoBeam

# A tool for individualized post hoc determination of the Beam F3 TMS target

---

> ### ⚠️ Important notice for users of v0.1.x
>
> Versions up to and including **v0.1.1** contained a defect in which
> `measurements/beam_f3_coordinates.csv` was overwritten with the **Beam_F3_irl**
> result whenever `x_f3_irl` and `y_f3_irl` were supplied — the irl coordinates
> were written under the `Beam_F3` / `Ref_Beam_F3` labels, with nothing to
> indicate the substitution. Runs using only `x_f3_geo` / `y_f3_geo` call the
> affected function once and are unaffected.
>
> **To check existing output:** `f3_coordinates.csv` is written before the irl
> calculation and still holds the correct geodesic coordinates. If it disagrees
> with `beam_f3_coordinates.csv` for a subject, that subject is affected.
> `f3_all_coordinates.csv`, `beam_f3_irl_coordinates.csv` and
> `ref_points_coordinates.csv` are assembled in memory and are correct in all
> versions.
>
> Fixed in v0.2.0. See [CHANGELOG.md](CHANGELOG.md) for details.

---

## Requirements

### Software dependencies

```
numpy
pandas
pyvista
meshio
pygeodesic
pymeshfix
open3d
```

Install with:

```
pip install numpy pandas pyvista meshio pygeodesic pymeshfix open3d
```

> **SimNIBS** (tested with SimNIBS 4.5) must be installed separately and used to generate the individual head mesh and EEG position files prior to running this script. See the [SimNIBS documentation](https://simnibs.github.io/simnibs/) for installation and head mesh generation.

### Required input files

For each subject, the script expects a SimNIBS `m2m_*` directory containing:

```
m2m_<subject_id>/
├── <subject_id>.msh              # SimNIBS head mesh file
└── eeg_positions/
    ├── EEG10-10_UI_Jurak_2007.csv   # 10-10 EEG electrode positions
    └── Fiducials.csv                # Fiducial positions (LPA, RPA, Nz, Iz)
```

These files are generated automatically by the SimNIBS `charm` segmentation pipeline.

---

## Usage

The main entry point is `run_all()`, which processes one or more subjects in
batch. Pass the root of your SimNIBS output tree as `base_path`; it must contain
one directory per subject, named `sub-001`, `sub-002`, ..., each holding an
`m2m_sub-XXX` directory. `run_all()` returns `(processed, failed)` and prints a
summary of any subjects that failed, so that a failure partway through a long
batch is not lost in the scrollback.

Importantly, in the current script iteration, a two-step run process is required. One initial run is performed to compute the anatomical distances, which are exported into the "measurements" directory. In the current script, these measurements need to be used in the online Beam F3 tool ("<https://clinicalresearcher.org/F3/>") to calculate x and y distances. These can then be used as input for a second run of the script which outputs the final Beam F3 target.

### Minimal example (single subject)

```python
from GeoBeam import main

main(
    mesh_file="/path/to/m2m_sub-001/sub-001.msh",
    m2m_dir="/path/to/m2m_sub-001",
    calculate_distances=True,
    calculate_vertex_Real=True,
    calculate_beam_f3=True,
    x_f3_geo=65.9,   # X_Beam distance from EB (mm)
    y_f3_geo=91.8,   # Y_Beam distance from Cz (mm)
    decimation_factor=0  # Set to 0 to disable mesh decimation
)
```

### Batch processing (multiple subjects)

```python
from GeoBeam import run_all

# Using subject-specific X/Y coordinates
subject_coords = {
    'sub-001': {'x_f3_geo': 65.9, 'y_f3_geo': 91.8, 'x_f3_irl': 66.8, 'y_f3_irl': 99.3},
    'sub-002': {'x_f3_geo': 67.2, 'y_f3_geo': 92.1, 'x_f3_irl': 68.1, 'y_f3_irl': 99.8},
}

processed, failed = run_all(
    min_index=1,
    max_index=2,
    base_path="/path/to/BIDS/derivatives/SimNIBS",
    subject_coordinates=subject_coords,
    decimation_factor=0,
    calculate_distances=True,
    calculate_beam_f3=True,
    calculate_vertex_Real=True,
    determine_geo_electrodes=True,
    geo_off_screen=True
)
```

See the commented examples at the bottom of the script for additional usage patterns, including mixed (subject-specific + default) coordinate assignment.

Subjects are processed independently, so batches can be parallelised trivially by
running several Python processes over disjoint index ranges. Note that each
process holds a full scalp mesh in memory, so the practical limit is available
RAM rather than core count — particularly with `decimation_factor=0`.

### Key parameters

| Parameter                  | Type    | Description                                                                                                          |
| -------------------------- | ------- | -------------------------------------------------------------------------------------------------------------------- |
| `mesh_file`                | `str`   | Path to the SimNIBS `.msh` head mesh file                                                                            |
| `m2m_dir`                  | `str`   | Path to the SimNIBS `m2m_*` directory                                                                                |
| `base_path`                | `str`   | (`run_all` only) Root of the SimNIBS output tree containing the per-subject directories. Required                    |
| `x_f3_geo`                 | `float` | X\_Beam distance (mm): distance from EB along the EB–T7 path                                                         |
| `y_f3_geo`                 | `float` | Y\_Beam distance (mm): distance from Vertex\_Real towards Off\_Mid\_F3                                               |
| `x_f3_irl`                 | `float` | Optional: in-real-life measured X\_Beam for comparison, can be used to test other coordinates in relation to Beam F3 |
| `y_f3_irl`                 | `float` | Optional: in-real-life measured Y\_Beam for comparison, can be used to test other coordinates in relation to Beam F3 |
| `decimation_factor`        | `float` | Mesh decimation (0–0.9); 0 disables decimation, 0.9 removes 90% of polygons                                          |
| `calculate_distances`      | `bool`  | Compute and export standard reference distances (Tr-Tr, Nz-Iz, Circumference)                                        |
| `calculate_beam_f3`        | `bool`  | Compute and export Beam\_F3 and Ref\_Beam\_F3 coordinates                                                            |
| `calculate_vertex_Real`    | `bool`  | Compute and export the geodesic vertex (Vertex\_Real). Required when `calculate_beam_f3=True`                        |
| `determine_geo_electrodes` | `bool`  | Compute geodesic-based 10-20 electrode positions for QA. Required to produce `f3_all_coordinates.csv`                |
| `geo_off_screen`           | `bool`  | Save visualizations as screenshots instead of interactive display                                                    |
| `visualize_beam_f3_plot`   | `bool`  | Render the Beam\_F3 path visualization (default `False`)                                                             |
| `beam_f3_off_screen`       | `bool`  | Save that visualization as a PNG instead of opening an interactive window (default `True`)                           |

---

## Outputs

All output files are written to `<m2m_dir>/measurements/` and `<m2m_dir>/visualizations/`.

| File                             | Contents                                                                                              |
| -------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `reference_distances.csv`        | Tr-Tr, Nz-Cz\_AP-Iz, Circ distances (mm)                                                              |
| `beam_f3_coordinates.csv`        | 3D coordinates of Beam\_F3 and Ref\_Beam\_F3                                                          |
| `beam_f3_irl_coordinates.csv`    | 3D coordinates of Beam\_F3\_irl and Ref\_Beam\_F3\_irl (if `x_f3_irl`/`y_f3_irl` supplied)            |
| `f3_coordinates.csv`             | 3D coordinates of Beam\_F3 (legacy name, retained for compatibility)                                  |
| `vertex_real_coordinates.csv`    | 3D coordinates of Vertex\_Real                                                                        |
| `vertex_real_distances.csv`      | Geodesic distances from Vertex\_Real to other reference points                                        |
| `geo_electrodes_coordinates.csv` | Geodesic-based electrode positions according to the 10-10 system (if `determine_geo_electrodes=True`) |
| `ref_points_coordinates.csv`     | Reference direction points for F3\_csv, F3\_Geo, Beam\_F3 and Beam\_F3\_irl                           |
| `f3_all_coordinates.csv`         | Comparison table of F3\_csv, F3\_Geo, Beam\_F3, Beam\_F3\_irl and their reference points              |
| `visualizations/*.png`           | Screenshot renders of key paths and points                                                            |

---

## Method Summary

The script implements the following steps:

1. **Scalp extraction**: The scalp surface (gmsh physical group 5) is extracted from the SimNIBS head mesh and repaired using PyMeshFix (with Open3D Poisson reconstruction as fallback for severely damaged meshes).
2. **Geodesic computation**: All surface distances are computed as true geodesic distances on the triangulated scalp mesh using the `pygeodesic` library ("<https://github.com/mhogg/pygeodesic>").
3. **Cz_AP localization**: The anatomically corrected midpoint (Cz_AP) along the Nasion–Inion arc is found by bisecting the total geodesic Nz–Iz path length.
4. **Vertex_Real localization**: The true geodesic midpoint of the Tragus–Tragus arc (LPA–RPA) through Cz_AP.
5. **Reference distances**: Standard cranial distances (Tr-Tr, Nz-Iz, Circumference) are computed geodesically.
6. **Beam F3 coordinates**: The stimulation site (Beam_F3) is located by projecting X_Beam from EB along the EB–T7 arc, then Y_Beam from the Vertex_Real towards the prior projection (Off_Mid_F3). A reference direction point (Ref_Beam_F3) indicating the coil orientation axis is found as the point on the Nz–Vertex_Real arc where the path to Beam_F3 intersects at 45°.

For full methodological details, validation results, and clinical context, refer to the accompanying paper.

---

## Version history

See [CHANGELOG.md](CHANGELOG.md). Tag `v0.1.1` marks the state of the code
released alongside the preprint.

---

## Citation

If you use this code in your work, please cite the accompanying paper:

> Schramm S, Ten Pas J, Calabrò D, Jakubetz J, Szillat M, Koti J, Huang M, Kim SH, Woletz M, Kirschke J, Hedderich DM, Sollmann N, Tik M, Vogelmann U. Post Hoc Localization of Beam F3 Stimulation Targets: An MRI-Derived Geodesic Approach for Refined TMS E-Field Simulations. *[Preprint — journal/DOI to be added upon publication.]*

Please also cite the underlying libraries:

- **SimNIBS**: Saturnino et al., *Brain and Human Body Modeling* 2019
- **pygeodesic**: Hogg & Kaszynski, [github.com/mhogg/pygeodesic](https://github.com/mhogg/pygeodesic)
- **EEG 10-10 positions**: Jurcak et al., *NeuroImage* 2007

---

## License

CC-BY-NC License. See `LICENSE` for details.
