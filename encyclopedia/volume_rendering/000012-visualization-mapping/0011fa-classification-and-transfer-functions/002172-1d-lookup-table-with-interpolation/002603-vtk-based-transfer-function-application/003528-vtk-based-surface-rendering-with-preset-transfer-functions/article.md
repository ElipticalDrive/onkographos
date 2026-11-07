# VTK-based Surface Rendering with Preset Transfer Functions

Also called: Label-based surface rendering, SurfaceActor rendering

## Overview
A surface rendering technique that converts label volumes to polygonal meshes and renders them using VTK's mapper/property pipeline, driven by JSON presets that define color transfer, scalar opacity, and lighting parameters [S1, unofficial]. It differs from ray-casting methods by generating explicit geometry (polygons) rather than projecting voxels, making it suitable for interactive exploration of segmented structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A label volume (scalar data where integer values represent distinct labels) and a preset configuration (JSON or structured parameters).
- Output: A `vtkActor` ready for insertion into a VTK renderer.

**Data Structures:**
The preset defines two primary transfer function structures:
1.  `color_transfer`: A list of 6-element lists `[scalar_value, r, g, b, ambient, diffuse]`.
2.  `scalar_opacity`: A list of 4-element lists `[scalar_value, opacity, ambient, diffuse]`.

**Parameters:**
- `alpha_scale`: Multiplier applied to preset alpha values before clamping. Default: `1.0` [S1, unofficial].
- `interpolation`: Shading interpolation mode string. Valid values: `'phong'`, `'flat'`, `'gouraud'` [S1, unofficial].
- `scalar_range`: Range for uncertainty preset color transfer. Default: `[0., 1.0e-2]` [S1, unofficial].

**Steps:**
1.  **Load Volume:** Load the input label volume. If the input is a NumPy array, convert it to a VTK volume with spacing `[1, 1, 1]` and origin `[0, 0, 0]` [S1, unofficial].
2.  **Initialize Lookup Table:** Create a `vtkLookupTable` with the number of labels equal to the colormap length. Iterate over colormap entries to set table values. Apply the `alpha_scale` multiplier to the alpha component and clamp the result to `[0, 1]`:
    $$ a = \min(\max(a \cdot \text{alpha\_scale}, 0.), 1.) $$
    [S1, unofficial].
3.  **Configure Mapper:** Create a `vtkPolyDataMapper`. Assign the surface mesh as input. Enable scalar visibility. Attach the lookup table. Set the scalar range to `(1, n_label)` where `n_label` is the number of labels. Call `Update()` on the mapper [S1, unofficial].
4.  **Configure Property:** Create a `vtkProperty`. Set lighting coefficients (ambient, diffuse, specular, specular_power) from the preset. Set the interpolation mode based on the `interpolation` string. Toggle shading and enable backface culling [S1, unofficial].
5.  **Create Actor:** Instantiate a `vtkActor`. Attach the mapper and property. Apply any user-defined transform. Return the actor [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import vtk
import numpy as np

def create_surface_actor(volume_data, preset, alpha_scale=1.0, interpolation='gouraud'):
    """
    Creates a VTK actor for surface rendering based on a label volume and preset.
    
    Args:
        volume_data: NumPy array or VTK image data representing label volume.
        preset: Dictionary containing 'color_transfer' and 'scalar_opacity' lists.
        alpha_scale: Multiplier for alpha values.
        interpolation: Shading mode ('phong', 'flat', 'gouraud').
        
    Returns:
        vtkActor instance.
    """
    # Step 1: Load/Convert Volume
    # Note: The source implies conversion from numpy to VTK volume if needed.
    # For surface rendering, we assume a mesh (vtkPolyData) is generated 
    # from the volume (e.g., via marching cubes) prior to this function 
    # or handled by the caller. The source code snippet shows 'surface' 
    # being passed to the mapper. We assume 'surface' is the vtkPolyData.
    # If volume_data is numpy, it must be converted to a mesh first.
    # This step is a placeholder for the mesh generation logic not detailed in the snippet.
    surface = generate_mesh_from_volume(volume_data) # [G] Placeholder for mesh generation
    
    # Step 2: Initialize Lookup Table
    # Determine number of labels. Assuming labels are 1..N.
    n_labels = int(np.max(volume_data)) if np.max(volume_data) > 0 else 256 # [G] Default fallback
    
    lut = vtk.vtkLookupTable()
    lut.SetNumberOfTableValues(n_labels + 1) # Index 0 is background, 1..N are labels
    
    # Set background (index 0)
    lut.SetTableValue(0, 0.0, 0.0, 0.0, 0.0)
    
    # Iterate over colormap entries from preset
    # The source snippet uses a fixed 256-entry cmap loop. 
    # Here we map preset entries to labels.
    for label_idx in range(1, n_labels + 1):
        # Find preset entry for this label
        # Simplified: assume preset has entries for each label or a continuous map
        # The source snippet uses 'cmap' from matplotlib. 
        # We will use the preset's color_transfer if available, else a default.
        
        # Extract color from preset or default
        # Assuming preset['color_transfer'] is a list of [val, r, g, b, amb, diff]
        # For label-based, we might map label index to a color.
        # The source snippet sets alpha with scaling.
        
        # Fallback to a simple colormap if preset is not fully detailed for labels
        # Using the source's logic: iterate and set values.
        
        # Let's assume we have a color for this label.
        # In the source, it uses cm.get_cmap. We will simulate setting values.
        
        # For the sake of the algorithm, we assume we can retrieve r,g,b for label i
        r, g, b = 0.5, 0.5, 0.5 # [G] Default color
        a = 1.0 # [G] Default alpha
        
        # Apply alpha scale and clamp
        a = min(max(a * alpha_scale, 0.0), 1.0)
        
        lut.SetTableValue(label_idx, r, g, b, a)
        
    lut.Build()
    
    # Step 3: Configure Mapper
    mapper = vtk.vtkPolyDataMapper()
    mapper.SetInputData(surface)
    mapper.SetScalarVisibility(1)
    mapper.SetLookupTable(lut)
    
    # Set scalar range to (1, n_label)
    mapper.SetScalarRange(1, n_labels)
    
    mapper.Update()
    
    # Step 4: Configure Property
    prop = vtk.vtkProperty()
    
    # Set lighting coefficients from preset
    # Default values if not in preset
    ambient = 0.0
    diffuse = 1.0
    specular = 0.0
    specular_power = 10.0
    
    prop.SetAmbient(ambient)
    prop.SetDiffuse(diffuse)
    prop.SetSpecular(specular)
    prop.SetSpecularPower(specular_power)
    
    # Set interpolation mode
    if interpolation == 'phong':
        prop.InterpolateScalarsBeforeMappingOn() # [G] Approximation for Phong
    elif interpolation == 'flat':
        prop.SetRepresentationToWireframe() # [G] Flat shading often implies no interpolation or wireframe
    elif interpolation == 'gouraud':
        prop.InterpolateScalarsBeforeMappingOff() # [G] Gouraud is default
        
    prop.ShadingOn()
    prop.BackfaceCullingOn()
    
    # Step 5: Create Actor
    actor = vtk.vtkActor()
    actor.SetMapper(mapper)
    actor.SetProperty(prop)
    
    # Apply user transform if present (not shown in snippet, but mentioned in steps)
    # transform = ... # [G] User transform handling
    
    return actor

def generate_mesh_from_volume(volume_data):
    """
    Placeholder for mesh generation (e.g., Marching Cubes).
    """
    # This is a complex operation not detailed in the source snippet.
    # The source assumes 'surface' is already available.
    raise NotImplementedError("Mesh generation logic is external to this method's core definition.")
```

## Subtopics

### VTK-Based Transfer Function Application
The parent method providing the infrastructure for applying color and opacity transfer functions using VTK classes. Choose this when you need to integrate surface rendering with the broader VTK-based classification pipeline. [Link](3725-vtk-based-transfer-function-application.md)

## Operating characteristics
- **Time Complexity**: Dependent on the mesh generation step (e.g., Marching Cubes is $O(N)$ where $N$ is the number of voxels) and the VTK rendering pipeline overhead. The lookup table construction is $O(L)$ where $L$ is the number of labels.
- **Memory Complexity**: Proportional to the size of the generated polygonal mesh and the lookup table.
- **Strengths**: Fast interactive rendering for static or slowly changing geometries; standard VTK integration; clear separation of geometry and appearance.
- **Limitations**: Requires explicit mesh generation; less effective for semi-transparent internal structures compared to ray casting; quality depends on mesh resolution.
- **When to Use**: When rendering segmented label volumes as solid surfaces with specific lighting and shading requirements.
- **When Not to Use**: When volumetric transparency or internal structure visualization is required.

## Background
Originated in the context of VTK-based volume rendering tools, specifically as implemented in the `pyvrender` library by yuta-hi [S1, unofficial]. It leverages VTK's standard actor/mapper paradigm to provide a surface-based alternative to ray-casting.

## Sources
[S1] Volume Renderer with VTK, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
