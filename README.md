[README.md](https://github.com/user-attachments/files/29177678/README.md)


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

```bash
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

The main entry point is `run_all()`, which processes one or more subjects in batch. Edit the `base_path` variable inside `run_all()` to point to your data directory, then call the function from the Python interpreter or by appending a call at the end of the script.
Importantly, in the current script iteration, a two-step run process is required. One initial run is performed to compute the anatomical distances, which are exported into the "measurements" directory. In the current script, these measurements need to be used in the online Beam F3 tool ("https://clinicalresearcher.org/F3/") to calculate x and y distances. These can then be used as input for a second run of the script which outputs the final Beam F3 target.

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

run_all(
    min_index=1,
    max_index=2,
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

### Key parameters

| Parameter | Type | Description |
|---|---|---|
| `mesh_file` | `str` | Path to the SimNIBS `.msh` head mesh file |
| `m2m_dir` | `str` | Path to the SimNIBS `m2m_*` directory |
| `x_f3_geo` | `float` | X_Beam distance (mm): distance from EB along the EB–T7 path |
| `y_f3_geo` | `float` | Y_Beam distance (mm): distance from Vertex_Real towards Off_Mid_F3 |
| `x_f3_irl` | `float` | Optional: in-real-life measured X_Beam for comparison, can be used to test other coordinates in relation to Beam F3 |
| `y_f3_irl` | `float` | Optional: in-real-life measured Y_Beam for comparison, can be used to test other coordinates in relation to Beam F3 |
| `decimation_factor` | `float` | Mesh decimation (0–0.9); 0 disables decimation, 0.9 removes 90% of polygons |
| `calculate_distances` | `bool` | Compute and export standard reference distances (Tr-Tr, Nz-Iz, Circumference, R1, R2) |
| `calculate_beam_f3` | `bool` | Compute and export Beam_F3 and Ref_Beam_F3 coordinates |
| `calculate_vertex_Real` | `bool` | Compute and export the geodesic vertex (Vertex_Real) |
| `determine_geo_electrodes` | `bool` | Compute geodesic-based 10-20 electrode positions for QA |
| `geo_off_screen` | `bool` | Save visualizations as screenshots instead of interactive display |

---

## Outputs

All output files are written to `<m2m_dir>/measurements/` and `<m2m_dir>/visualizations/`.

| File | Contents |
|---|---|
| `reference_distances.csv` | Tr-Tr, Nz-Cz_AP-Iz, Circ, R1, R2, and Vertex_Real distances (mm) |
| `beam_f3_coordinates.csv` | 3D coordinates of Beam_F3 and Ref_Beam_F3 |
| `vertex_real_coordinates.csv` | 3D coordinates of Vertex_Real, Cz_AP, LPA, RPA |
| `vertex_real_distances.csv` | Geodesic distances from Vertex_Real to LPA and RPA |
| `geo_electrodes_coordinates.csv` | Geodesic-based electrode positions (if `determine_geo_electrodes=True`) |
| `f3_all_coordinates.csv` | Comparison table of F3_csv, F3_Geo, Beam_F3, Beam_F3_irl and their reference points |
| `visualizations/*.png` | Screenshot renders of key paths and points |

---

## Method Summary

The script implements the following steps:

1. **Scalp extraction**: The scalp surface (gmsh physical group 5) is extracted from the SimNIBS head mesh and repaired using PyMeshFix (with Open3D Poisson reconstruction as fallback for severely damaged meshes).
2. **Geodesic computation**: All surface distances are computed as true geodesic distances on the triangulated scalp mesh using the `pygeodesic` library (Mitchell–Mount–Papadimitriou algorithm).
3. **Cz_AP localization**: The anatomically corrected midpoint (Cz_AP) along the Nasion–Inion arc is found by bisecting the total geodesic Nz–Iz path length.
4. **Vertex_Real localization**: The true geodesic midpoint of the Tragus–Tragus arc (LPA–RPA) through Cz_AP.
5. **Reference distances**: Standard cranial distances (Tr-Tr, Nz-Iz, Circumference) are computed geodesically.
6. **Beam F3 coordinates**: The stimulation site (Beam_F3) is located by projecting X_Beam from EB along the EB–T7 arc, then Y_Beam from the Vertex_Real towards the prior projection (Off_Mid_F3). A reference direction point (Ref_Beam_F3) indicating the coil orientation axis is found as the point on the Nz–Vertex_Real arc where the path to Beam_F3 intersects at 45°.

For full methodological details, validation results, and clinical context, refer to the accompanying paper.

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
