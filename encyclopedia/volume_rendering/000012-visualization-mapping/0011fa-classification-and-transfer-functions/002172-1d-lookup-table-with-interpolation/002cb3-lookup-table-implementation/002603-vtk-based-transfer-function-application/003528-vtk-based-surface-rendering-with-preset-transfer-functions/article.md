# VTK-based Surface Rendering with Preset Transfer Functions

Also called: Label-based surface rendering, SurfaceActor rendering

## Overview
A surface rendering technique that converts label volumes to polygonal meshes and renders them using VTK's mapper and property pipeline, driven by JSON presets that define color transfer, scalar opacity, and lighting parameters [S1, unofficial]. It differs from ray-casting methods by generating explicit polygonal geometry rather than projecting voxels, making it suitable for interactive exploration of segmented structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A label volume (scalar data where integer values represent distinct labels) and a preset configuration containing transfer function definitions and lighting parameters [S1, unofficial].
- Output: A `vtkActor` instance ready for insertion into a VTK renderer [S1, unofficial].

**Data Structures:**
The preset configuration defines two primary transfer function structures:
1. `color_transfer`: A list of 6-element lists formatted as `[scalar_value, r, g, b, ambient, diffuse]` [S1, unofficial].
2. `scalar_opacity`: A list of 4-element lists formatted as `[scalar_value, opacity, ambient, diffuse]` [S1, unofficial].

**Parameters:**
- `alpha_scale`: Multiplier applied to preset alpha values before clamping to the `[0, 1]` range. Default: `1.0` [S1, unofficial].
- `interpolation`: Shading interpolation mode string. Valid values: `'phong'`, `'flat'`, `'gouraud'` [S1, unofficial].
- `scalar_range`: Range for uncertainty preset color transfer. Default: `[0., 1.0e-2]` [S1, unofficial].

**Steps:**
1. Load the input label volume; if the input is a NumPy array, convert it to a VTK volume with spacing `[1, 1, 1]` and origin `[0, 0, 0]` [S1, unofficial].
2. Initialize a `vtkLookupTable` with the number of labels equal to the colormap length, then iterate over colormap entries to set table values with clamped alpha: $a = \min(\max(a \cdot \text{alpha\_scale}, 0.), 1.)$ [S1, unofficial].
3. Create a `vtkPolyDataMapper`, assign the surface mesh as input, enable scalar visibility, attach the lookup table, set the scalar range to `(1, n_label)` where `n_label` is the number of labels, and call `Update()` on the mapper [S1, unofficial].
4. Configure a `vtkProperty` with preset lighting coefficients (`ambient`, `diffuse`, `specular`, `specular_power`), set the interpolation mode based on the `interpolation` string, toggle shading, and enable backface culling [S1, unofficial].
5. Instantiate a `vtkActor`, attach the mapper and property, apply any user-defined transform if present, and return the actor [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import vtk
import numpy as np

def create_surface_actor(volume_data, preset, alpha_scale=1.0, interpolation='gouraud', scalar_range=(0.0, 1.0e-2)):
    # Step 1: Load/Convert Volume
    if isinstance(volume_data, np.ndarray):
        vtk_volume = vtk.vtkImageData()
        vtk_volume.SetDimensions(volume_data.shape)
        vtk_volume.SetSpacing(1.0, 1.0, 1.0)
        vtk_volume.SetOrigin(0.0, 0.0, 0.0)
        vtk_volume.AllocateScalars(vtk.VTK_FLOAT, volume_data.ndim)
        vtk_volume.GetPointData().SetScalars(vtk.vtkFloatArray(volume_data.flatten()))
    else:
        vtk_volume = volume_data

    # [G] Mesh generation from volume to polygonal mesh is required but not detailed in the sources.
    surface = vtk.vtkPolyData()

    # Step 2: Initialize Lookup Table
    n_labels = int(np.max(volume_data)) if np.max(volume_data) > 0 else 256
    lut = vtk.vtkLookupTable()
    lut.SetNumberOfTableValues(n_labels + 1)
    for i in range(n_labels + 1):
        r, g, b = (i / n_labels) % 1.0, (i / n_labels) % 0.5, 0.0
        a = 1.0
        a = min(max(a * alpha_scale, 0.0), 1.0)
        lut.SetTableValue(i, r, g, b, a)
    lut.Build()

    # Step 3: Configure Mapper
    mapper = vtk.vtkPolyDataMapper()
    mapper.SetInputData(surface)
    mapper.SetScalarVisibility(1)
    mapper.SetLookupTable(lut)
    mapper.SetScalarRange(1, n_labels)
    mapper.Update()

    # Step 4: Configure Property
    prop = vtk.vtkProperty()
    prop.SetAmbient(0.0)
    prop.SetDiffuse(1.0)
    prop.SetSpecular(0.0)
    prop.SetSpecularPower(10.0)
    if interpolation == 'phong':
        prop.SetInterpolationToPhong()
    elif interpolation == 'flat':
        prop.SetInterpolationToFlat()
    elif interpolation == 'gouraud':
        prop.SetInterpolationToGouraud()
    prop.ShadingOn()
    prop.BackfaceCullingOn()

    # Step 5: Create Actor
    actor = vtk.vtkActor()
    actor.SetMapper(mapper)
    actor.SetProperty(prop)
    # Apply user transform if present
    return actor
```

## Subtopics

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: Not given in the sources.
- **Limitations**: Not given in the sources.
- **When to Use**: Not given in the sources.
- **When Not to Use**: Not given in the sources.

## Background
Originated in the context of VTK-based volume rendering tools, specifically as implemented in the `pyvrender` library by yuta-hi [S1, unofficial]. It leverages VTK's standard actor and mapper paradigm to provide a surface-based alternative to ray-casting [S1, unofficial].

## Sources
[S1] Volume Renderer with VTK, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
