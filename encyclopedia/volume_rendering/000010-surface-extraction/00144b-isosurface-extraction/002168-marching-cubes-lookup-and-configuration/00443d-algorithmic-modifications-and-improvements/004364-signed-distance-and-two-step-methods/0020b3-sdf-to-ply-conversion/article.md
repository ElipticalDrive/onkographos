# SDF to PLY Conversion

Also called: Implicit to Mesh Conversion, Marching Cubes from SDF

## Overview
This method converts a Signed Distance Field (SDF) defined on a structured 3D grid into a polygonal triangle mesh (PLY format) by sampling the implicit function and applying the Marching Cubes algorithm [S1, S2, unofficial]. It differs from its siblings by explicitly handling the transformation from voxel indices to world coordinates and managing memory during implicit function evaluation via chunking [S2, unofficial]. Unlike [Two-Step Surface Reconstruction](7364-two-step-surface-reconstruction-method.md) which may use narrow-band filtering, this approach evaluates the full grid or dense chunks to ensure topological correctness at the zero-level set [S2, unofficial].

## How it works
**Inputs:**
*   `implicit_fn`: A callable that computes the SDF value $f(x, y, z)$ for any 3D coordinate [S2, unofficial].
*   `grid_size`: Integer resolution of the grid, default 128 [S2, unofficial].
*   `scale`: Spatial extent of the grid, default 0.5 [S2, unofficial].
*   `thresh`: Threshold for the isosurface, default 0.0 [S2, unofficial].
*   `chunk_size`: Number of points to evaluate per batch to limit memory, default 262144 [S2, unofficial].
*   `voxel_grid_origin`: Offset added to vertex coordinates to shift from voxel grid space to world/camera space [S1, unofficial].

**Outputs:**
*   A PLY file containing vertex positions and face indices [S1, unofficial].

**Data Structures:**
1.  **Grid Points:** A tensor/array of shape $(N, 3)$ where $N = (grid\_size + 1)^3$, containing world coordinates [S2, unofficial].
2.  **SDF Values:** A 3D array of shape $(grid\_size + 1, grid\_size + 1, grid\_size + 1)$ of float32 values [S2, unofficial].
3.  **Vertices:** NumPy structured array with fields `x`, `y`, `z` of type `float32` ('f4') [S1, unofficial].
4.  **Faces:** NumPy structured array with field `vertex_indices` of type `int32` ('i4') and shape $(3,)$ [S1, unofficial].

**Steps:**

1.  **Grid Generation:** Create a 3D grid of points by sampling `grid_size + 1` points linearly spaced between `-scale` and `scale` for X, Y, and Z axes. Stack these into a grid tensor $G$ [S2, unofficial].
    $$ G = \text{meshgrid}(\text{linspace}(-scale, scale, grid\_size + 1)) $$

2.  **SDF Evaluation (Chunked):** Evaluate the implicit function $f$ on the grid points. To manage memory, process points in chunks of size `chunk_size` [S2, unofficial].
    For each chunk $i$ from $0$ to $N$ with step `chunk_size`:
    $$ SDF_{chunk} = implicit\_fn.get\_distance(G[i : i + chunk\_size]) $$
    Store results in a flat array $SDF_{flat}$ [S2, unofficial].

3.  **Reshape SDF:** Reshape the flat SDF array back to 3D grid dimensions:
    $$ SDF_{3D} = SDF_{flat}.view(grid\_size + 1, grid\_size + 1, grid\_size + 1) $$
    [S2, unofficial].

4.  **Marching Cubes Extraction:** Run the Marching Cubes algorithm (`skimage.measure.marching_cubes_lewiner`) on $SDF_{3D}$ with `level=thresh` and `spacing=[voxel\_size, voxel\_size, voxel\_size]` [S1, unofficial].
    *   `voxel_size` is derived from `scale` and `grid_size`:
        $$ voxel\_size = \frac{2 \times scale}{grid\_size} $$
    *   Output: `vertices`, `faces`, `normals`, `values` [S1, unofficial].

5.  **Coordinate Transformation:** Transform vertex coordinates from voxel grid space to world/camera space.
    *   If `voxel_grid_origin` is provided, add it to each vertex coordinate [S1, unofficial].
    *   Alternatively, normalize vertices to the range `[-scale, scale]` if the Marching Cubes output is in index space:
        $$ v_{world} = \left( \frac{v_{index}}{grid\_size} - 0.5 \right) \times 2 \times scale $$
        [S2, unofficial].

6.  **PLY Construction:**
    *   Construct a NumPy structured array for vertices with fields `x`, `y`, `z` of type `float32` [S1, unofficial].
    *   Construct a NumPy structured array for faces with field `vertex_indices` of type `int32` and shape $(3,)$ [S1, unofficial].
    *   Create PLY elements using `plyfile.PlyElement.describe` and write to the output file [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from skimage import measure
import plyfile

def sdf_to_ply(implicit_fn, grid_size=128, scale=0.5, thresh=0.0, chunk_size=262144, voxel_grid_origin=None):
    # Step 1: Grid Generation
    axis = np.linspace(-scale, scale, grid_size + 1)
    x, y, z = np.meshgrid(axis, axis, axis, indexing='ij')
    grid_points = np.stack((x, y, z), axis=-1).reshape(-1, 3)
    
    # Step 2: SDF Evaluation (Chunked)
    num_points = grid_points.shape[0]
    sdf_flat = np.zeros(num_points, dtype=np.float32)
    
    for start in range(0, num_points, chunk_size):
        end = min(start + chunk_size, num_points)
        chunk = grid_points[start:end]
        sdf_chunk = implicit_fn.get_distance(chunk)
        if not isinstance(sdf_chunk, np.ndarray):
            sdf_chunk = np.array(sdf_chunk)
        sdf_flat[start:end] = sdf_chunk
        
    # Step 3: Reshape SDF
    sdf_3d = sdf_flat.reshape(grid_size + 1, grid_size + 1, grid_size + 1)
    
    # Step 4: Marching Cubes Extraction
    voxel_size = (2 * scale) / grid_size
    verts, mc_faces, normals, values = measure.marching_cubes_lewiner(
        sdf_3d, level=thresh, spacing=(voxel_size, voxel_size, voxel_size)
    )
    
    # Step 5: Coordinate Transformation
    if voxel_grid_origin is not None:
        verts += voxel_grid_origin
    # Note: marching_cubes_lewiner with spacing returns world coordinates.
    # If spacing was not used or indices are expected, normalize:
    # verts = (verts / grid_size - 0.5) * 2 * scale
    
    # Step 6: PLY Construction
    vertices = np.zeros(len(verts), dtype=[('x', 'f4'), ('y', 'f4'), ('z', 'f4')])
    vertices['x'] = verts[:, 0]
    vertices['y'] = verts[:, 1]
    vertices['z'] = verts[:, 2]
    
    faces_struct = np.zeros(len(mc_faces), dtype=[('vertex_indices', 'i4', (3,))])
    faces_struct['vertex_indices'] = mc_faces
    
    vertex_element = plyfile.PlyElement.describe(vertices, 'vertex')
    face_element = plyfile.PlyElement.describe(faces_struct, 'face')
    plyfile.PlyData([vertex_element, face_element]).write('output.ply')
```

**Edge Cases and Pitfalls:**
*   **Memory Overflow:** Evaluating the entire grid at once can exceed GPU/CPU memory for large `grid_size` [S2, unofficial].
*   **Precision:** Use `float32` for SDF values and vertices to balance precision and memory, but be aware of aliasing artifacts at low resolutions [S1, S2, unofficial].
*   **Voxel Origin:** Incorrect handling of `voxel_grid_origin` leads to meshes displaced from the expected world coordinates [S1, unofficial].

## Subtopics

### [SDF to Mesh Extraction](8429-sdf-to-mesh-extraction.md)
Extracts a triangle mesh from a signed distance field (SDF) grid at a specified isosurface value. It builds on [SDF to PLY Conversion](2343-sdf-to-ply-conversion.md) by converting the volumetric representation into a polygonal surface [G]. Unlike optimized extraction methods that may simplify topology, this approach performs a direct extraction. Link: [SDF to Mesh Extraction](8429-sdf-to-mesh-extraction.md).

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of grid points, dominated by the implicit function evaluation [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the SDF grid and grid points [S2, unofficial].
*   **Performance:** Depends heavily on the cost of `implicit_fn` [S2, unofficial].
*   **Strengths:** Simple to implement; leverages robust Marching Cubes implementations [S1, unofficial].
*   **Limitations:** Fixed resolution grid may not capture fine details efficiently; aliasing artifacts at low resolutions [S2, unofficial].
*   **When to Use:** When the SDF is available on a regular grid and high performance is not critical [S2, unofficial].

## Background
The method originates from the need to visualize implicit surfaces defined by SDFs in computer graphics and 3D reconstruction [S1, S2, unofficial]. It builds on the classic Marching Cubes algorithm [S1, unofficial]. The specific implementation details, such as chunked evaluation and PLY output, are derived from practical implementations in neural surface reconstruction pipelines [S2, unofficial].

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
