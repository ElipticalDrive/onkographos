# Trimmed Voxel Lists


## Overview
Trimmed Voxel Lists is a surface extraction algorithm that represents an iso-surface using 3D points as geometric primitives, projecting exactly one point per visible surface cell to ensure image coverage without recursive subdivision [S1, S2, unofficial]. It differs from Dividing Cubes by avoiding subdivision; instead, it allows a single projected point to span up to three pixels on the image plane to guarantee coverage [S1, S2, unofficial].

## How it works
**Inputs:**
- Volumetric data grid.
- Iso-value threshold.
- Camera parameters (view and projection matrices).

**Outputs:**
- A list of visible surface points, each associated with a 3D position and a normal vector.

**Data Structures:**
- A list of surface points. Each entry contains:
  - `position`: 3D coordinates $(x, y, z)$ of the point in world space.
  - `normal`: 3D normal vector $(n_x, n_y, n_z)$ at that position.

**Steps:**
1. **Identify Visible Surface Cells:** Iterate through the volume grid to identify cells intersected by the iso-surface. Filter out cells that are not visible from the current camera viewpoint [S2, unofficial].
2. **Select One Point Per Cell:** For each visible surface cell, compute a single representative 3D point on the iso-surface within that cell. The method does not subdivide the cell [S1, S2, unofficial].
3. **Compute Normal:** Calculate the surface normal at the selected point, typically using the gradient of the scalar field at that location [S1, unofficial].
4. **Project to Image Plane:** Project the 3D point and its normal onto the 2D image plane. The projection is designed such that the point covers up to three pixels to ensure the surface is rendered without gaps [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trimmed_voxel_lists(volume_data, iso_value, camera):
    surface_points = []
    
    # Step 1: Identify visible surface cells
    # Iterate through all cells in the volume grid
    for cell in volume_data.cells:
        if not is_surface_cell(cell, iso_value):
            continue
        if not is_visible(cell, camera):
            continue
            
        # Step 2: Select one point per cell
        # Compute a single point on the iso-surface within the cell
        point = compute_surface_point(cell, iso_value)
        
        # Step 3: Compute normal
        # Calculate the gradient at the point to get the normal
        normal = compute_normal(cell, point)
        
        # Step 4: Project to image plane
        # Project the point to screen coordinates
        screen_pos = project_to_screen(point, camera)
        
        # The point is rendered to cover up to 3 pixels
        # This is handled by the renderer's point size or rasterization logic
        surface_points.append({
            'position': point,
            'normal': normal,
            'screen_pos': screen_pos
        })
        
    return surface_points

def is_surface_cell(cell, iso_value):
    # Check if the cell intersects the iso-surface
    # Typically involves checking voxel values against iso_value
    # Not given in the sources. [G]
    return False

def is_visible(cell, camera):
    # Check if the cell is front-facing or occluded
    # Not given in the sources. [G]
    return False

def compute_surface_point(cell, iso_value):
    # Compute a representative point on the iso-surface
    # Not given in the sources. [G]
    return Vector3(0, 0, 0)

def compute_normal(cell, point):
    # Compute the gradient at the point
    # Not given in the sources. [G]
    return Vector3(0, 0, 0)

def project_to_screen(point, camera):
    # Project 3D point to 2D screen coordinates
    # Not given in the sources. [G]
    return Vector2(0, 0)
```

**Parameters:**
- Iso-value: The threshold for surface extraction.
- Camera: View and projection parameters.

**Edge Cases and Pitfalls:**
- **Coverage:** The method relies on projecting a single point to cover up to three pixels. If the point size or rasterization logic is not configured correctly, gaps may appear in the rendered surface [S1, S2, unofficial].
- **Accuracy:** Since only one point per cell is used, the approximation of the surface may be less accurate than methods that subdivide cells [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of visible surface cells, as it avoids recursive subdivision [S1, S2, unofficial].
- **Memory Complexity:** Proportional to the number of visible surface points, which is typically less than the total number of cells [S2, unofficial].
- **Strengths:** Fast rendering due to no subdivision; suitable for interactive visualization [S1, S2, unofficial].
- **Limitations:** Lower accuracy compared to subdivision-based methods; potential for gaps if projection coverage is not properly handled [S2, unofficial].
- **When to Use:** Interactive visualization where speed is prioritized over high-fidelity surface representation [S1, S2, unofficial].
- **When Not to Use:** Applications requiring high-precision surface geometry [S2, unofficial].

## Background
The Trimmed Voxel Lists method was introduced as an alternative to Dividing Cubes, which subdivides cells to ensure each subcell projects onto a single pixel [S1, S2, unofficial]. The method was proposed to address the computational cost of subdivision while maintaining image coverage [S1, S2, unofficial]. It is used in volume visualization for efficient surface rendering [S1, S2, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
