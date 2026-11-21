# VTK Inner Ear Visualization (Python)

Interactive medical visualization tool built with **VTK** and **PyQt5**.
Case study: **inner ear anatomy** — volume data (**.nrrd**) + segmented 3D structures (**.vtk**).

## Features
- Load raw 3D image volume from **NRRD**
- Load multiple **VTK mesh** models from a folder (inner ear structures)
- 3 orthogonal slice views: **sagittal / coronal / axial**
- Keyboard slice control:
  - Left/Right → sagittal
  - Up/Down → coronal
  - `x` / `z` → axial
- Opacity control panel (PyQt5 sliders, 10–100%) for each structure
- Distinct colors for each mesh + semi-transparent default opacity (0.8)

## Data source
Inner ear dataset from OpenAnatomy:
https://www.openanatomy.org/atlas-pages/atlas-spl-inner-ear.html
