# VoxelGrid from Mesh (Fill Method)
## Overview
This method converts a triangular mesh into a dense voxel grid by first computing surface occupancy and then filling internal voids using morphological operations [S1, unofficial]. It differs from standard surface-only voxelization by ensuring the resulting volume is a solid, watertight representation, which is critical for differentiable rendering pipelines where internal density consistency is required [S1, unofficial].
## How it works
**Inputs and outputs:**
- Input: A mesh (vertices and faces) and a target resolution $(N_x, N_y, N_z)$.
- Output: A 3D boolean array `occ` of shape $(N_x, N_y, N_z)$, where `True` indicates a filled voxel.

**Data structures:**
- `occ`: A 3D NumPy array of boolean values.

**Steps:**
1. **Validate Bounds**: The method assumes the mesh is normalized to the unit cube $[-0.5, 0.5]^3$. If any bound of the mesh exceeds this range, raise a `ValueError` [S1, unofficial].
2. **Compute Surface Occupancy**: Voxelize the mesh surface to determine which voxels intersect the mesh geometry. Let this result be `occ_surface` [S1, unofficial].
3. **Fill Holes**: Apply a binary morphological fill operation to `occ_surface` to fill all enclosed voids. The result is the final `occ` grid [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy import ndimage

def voxelize_fill(mesh, resolution):
    """
    Voxelize a mesh by filling holes in the surface voxel representation.
    
    Args:
        mesh: A mesh object with vertices and faces.
        resolution: Integer or tuple of integers (Nx, Ny, Nz).
        
    Returns:
        occ: A 3D boolean array of shape (Nx, Ny, Nz).
    """
    # Step 1: Validate bounds
    # Assuming mesh has a method or property to get bounding box
    bounds = mesh.get_bounds() # Returns min and max coordinates
    
    # Check if bounds are within [-0.5, 0.5]
    if np.any(np.abs(bounds) >= 0.5):
        raise ValueError("Bounds must be within [-0.5, 0.5]")
    
    # Step 2: Compute surface occupancy
    # This step typically involves ray casting or grid intersection tests
    # to determine which voxels are intersected by the mesh surface.
    # The implementation details of voxelize_surface are not provided in the source.
    occ_surface = voxelize_surface(mesh, resolution)
    
    # Step 3: Fill holes
    # Use binary fill holes to fill enclosed regions
    occ = ndimage.morphology.binary_fill_holes(occ_surface)
    
    return occ
```

**Parameters:**
- `resolution`: The number of voxels along each axis. Typical values depend on the desired level of detail, e.g., $64^3$, $128^3$, or $256^3$ [S1, unofficial].

**Edge cases and pitfalls:**
- **Out-of-bounds mesh**: The method explicitly checks for bounds outside $[-0.5, 0.5]$ and raises an error [S1, unofficial].
- **Non-watertight meshes**: If the mesh is not watertight, the fill operation may fill external space as well, depending on the implementation of `binary_fill_holes`. It is assumed the input mesh is closed [S1, unofficial].
- **Precision**: The validation step uses `np.abs(bounds) >= 0.5`, which may fail for meshes exactly on the boundary. Floating-point precision issues should be considered [S1, unofficial].

## Subtopics

### [Morphological Dilation](3346-morphological-dilation.md)
Expands foreground voxels to fill small holes and smooth boundaries. It operates on discrete binary masks, distinct from continuous offsetting.

## Operating characteristics
**Time and memory complexity:**
- Time: Dominated by the surface voxelization step, which is typically $O(N \cdot R^3)$ or better with acceleration structures, where $N$ is the number of triangles and $R$ is the resolution. The fill step is $O(R^3)$ [S1, unofficial].
- Memory: $O(R^3)$ for the boolean occupancy grid [S1, unofficial].

**Measured performance:**
- Not given in the sources.

**Scaling:**
- Scales linearly with the number of voxels for the fill step. Surface voxelization scales with mesh complexity and resolution [S1, unofficial].

**Strengths and limitations:**
- Strengths: Produces solid, watertight volumes suitable for differentiable rendering. Simple to implement using standard libraries [S1, unofficial].
- Limitations: Requires the mesh to be watertight for correct internal filling. Resolution-dependent [S1, unofficial].

**When to use it and when not:**
- Use when a solid, filled volume is required for rendering or simulation.
- Do not use if only surface representation is needed or if the mesh is not watertight [S1, unofficial].

## Background
This method is part of the differentiable volumetric rendering pipeline developed by the Autonomous Vision Group [S1, unofficial]. It is used to generate voxel grids from mesh inputs for training neural networks that predict 3D shapes [S1, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
