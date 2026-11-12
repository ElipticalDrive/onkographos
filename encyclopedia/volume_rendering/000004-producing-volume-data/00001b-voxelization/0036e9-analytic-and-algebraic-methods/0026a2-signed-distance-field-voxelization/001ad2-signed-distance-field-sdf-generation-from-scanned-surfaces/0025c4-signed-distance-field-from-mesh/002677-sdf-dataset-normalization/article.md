# SDF Dataset Normalization
Also called: SDF Mesh Loading
## Overview
SDF Dataset Normalization is a preprocessing step that transforms the vertex coordinates of an input 3D mesh so that the geometry fits within a standardized canonical space, typically the unit cube $[-1, 1]^3$ centered at the origin [S1, unofficial]. This method solves the problem of inconsistent scale and position in raw mesh data, which is critical for downstream Signed Distance Field (SDF) generation where grid resolution and sampling bounds depend on predictable input dimensions [S1, unofficial]. It differs from [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md) and [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md) by operating directly on the source mesh geometry prior to discretization, ensuring the input data is normalized before voxelization [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A path to a mesh file (e.g., STL, OBJ) and a mesh loading library (e.g., `trimesh`).
- Output: A normalized mesh object where vertices are scaled and translated to fit within the bounding box $[-1, 1]^3$.

**Data Structures:**
- `mesh`: An object containing vertex positions (`mesh.vertices`) and faces (`mesh.faces`).
- `vs`: A numpy array of shape $(N, 3)$ containing vertex coordinates.
- `vmin`, `vmax`: Vectors of shape $(3,)$ representing the minimum and maximum coordinates along each axis.
- `v_center`: A vector of shape $(3,)$ representing the center of the bounding box.

**Steps:**
1. Load the mesh from the given file path using a library such as `trimesh.load` with `force='mesh'` to ensure a single mesh object is returned [S1, unofficial].
2. Extract the vertex coordinates $v_s$ from the loaded mesh [S1, unofficial].
3. Calculate the minimum coordinate vector $v_{min}$ by taking the element-wise minimum of $v_s$ along axis 0 (the vertex index axis) [S1, unofficial].
   $$ v_{min} = \min(v_s, \text{axis}=0) $$
4. Calculate the maximum coordinate vector $v_{max}$ by taking the element-wise maximum of $v_s$ along axis 0 [S1, unofficial].
   $$ v_{max} = \max(v_s, \text{axis}=0) $$
5. Calculate the center of the bounding box $v_{center}$ as the midpoint between $v_{min}$ and $v_{max}$ [S1, unofficial].
   $$ v_{center} = \frac{v_{min} + v_{max}}{2} $$
6. Compute the scale factor $s$ to map the bounding box to the range $[-1, 1]$. The side length of the bounding box is $L = v_{max} - v_{min}$. The maximum extent is $\max(L)$. The scale is:
   $$ s = \frac{2}{\max(L)} $$
   [G] Note: The provided notes explicitly define the calculation of $v_{center}$ but do not explicitly state the final scaling and translation formula. However, standard normalization to $[-1, 1]$ requires subtracting the center and multiplying by the scale. The implementation must apply this transformation to all vertices.
7. Transform each vertex $v$ to $v'$:
   $$ v' = (v - v_{center}) \times s $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import trimesh
import numpy as np

def normalize_sdf_mesh(mesh_path: str) -> trimesh.Trimesh:
    # Load mesh
    mesh = trimesh.load(mesh_path, force='mesh')
    
    # Extract vertices
    vs = mesh.vertices
    
    # Calculate bounds
    vmin = np.min(vs, axis=0)
    vmax = np.max(vs, axis=0)
    
    # Calculate center
    v_center = (vmin + vmax) / 2.0
    
    # Calculate scale to fit in [-1, 1]
    # The bounding box size
    bbox_size = vmax - vmin
    # The maximum dimension determines the scale to fit the unit cube
    max_dim = np.max(bbox_size)
    
    # Avoid division by zero for degenerate meshes
    if max_dim == 0:
        return mesh
        
    scale = 2.0 / max_dim
    
    # Normalize vertices: translate to center, then scale
    normalized_vertices = (vs - v_center) * scale
    
    # Update mesh vertices
    mesh.vertices = normalized_vertices
    
    return mesh
```

**Parameters:**
- Target Range: $[-1, 1]$ for all axes [S1, unofficial].

**Edge Cases and Pitfalls:**
- Degenerate meshes with zero volume (e.g., flat planes or points) result in a zero bounding box dimension, causing division by zero during scaling. A guard clause is required [G].
- Non-manifold geometry or disconnected components are handled by the bounding box calculation, which encompasses all vertices regardless of connectivity [S1, unofficial].

## Subtopics

### [Mesh Normalization](3796-mesh-normalization.md)
General preprocessing to fit geometry in a canonical space. Chosen when normalizing raw mesh data before any voxelization or SDF generation.

### [SDF Grid Resampling from File](4878-sdf-grid-resampling-from-file.md)
Resampling an existing SDF grid. Chosen when the input is already a voxel grid rather than a mesh.

### [SDF Training Data Resampling](13279-sdf-training-data-resampling.md)
Generating training points for neural networks. Chosen when the goal is neural rendering training data rather than a static SDF dataset.

## Operating characteristics
- Time Complexity: $O(N)$ where $N$ is the number of vertices, as it requires a single pass to find min/max and one pass to transform vertices.
- Memory Complexity: $O(N)$ to store the vertex array.
- Strengths: Ensures consistent input scale for SDF generation, preventing resolution mismatches.
- Limitations: Does not handle mesh topology or cleaning, only coordinate transformation.

## Background
This method is part of the data preparation pipeline for neural implicit representations, specifically for generating Signed Distance Fields from mesh inputs. It is implemented in projects such as IDDR-NGP [S1, unofficial].

## Sources
[S1] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
