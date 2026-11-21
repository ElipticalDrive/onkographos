# SDF to PLY Conversion

Also called: Implicit to Mesh Conversion, Marching Cubes from SDF

## Overview
This method converts a Signed Distance Field (SDF) defined on a structured 3D grid into a polygonal triangle mesh (PLY format) by sampling the implicit function and applying the Marching Cubes algorithm [S1, S2, S3, unofficial]. It differs from its siblings by explicitly handling the transformation from voxel indices to world coordinates and managing memory during implicit function evaluation, often via chunking [S2, unofficial]. Unlike [Two-Step Surface Reconstruction](7364-two-step-surface-reconstruction-method.md) which may use narrow-band filtering, this approach typically evaluates the full grid or a dense chunk to ensure topological correctness at the zero-level set [S2, unofficial].

## How it works
**Inputs:**
*   `implicit_fn`: A callable or object that computes the SDF value $f(x, y, z)$ for any 3D coordinate.
*   `grid_size`: Integer resolution of the grid (e.g., 128) [S2, unofficial].
*   `scale`: Spatial extent of the grid (e.g., 0.5) [S2, unofficial].
*   `thresh`: Threshold for the isosurface, typically 0.0 [S2, unofficial].
*   `chunk_size`: Number of points to evaluate per batch to limit memory (e.g., 262144) [S2, unofficial].
*   `voxel_grid_origin`: Optional offset to transform voxel space to world space [S1, unofficial].

**Outputs:**
*   A PLY file containing vertex positions and face indices.

**Data Structures:**
1.  **Grid Points:** A tensor of shape $(N, 3)$ where $N = (grid\_size + 1)^3$, containing world coordinates.
2.  **SDF Values:** A 3D array of shape $(grid\_size + 1, grid\_size + 1, grid\_size + 1)$ of float32 values [S2, unofficial].
3.  **Vertices:** NumPy structured array with fields `x`, `y`, `z` of type `float32` ('f4') [S1, unofficial].
4.  **Faces:** NumPy structured array with field `vertex_indices` of type `int32` ('i4') and shape $(3,)$ [S1, unofficial].

**Steps:**

1.  **Grid Generation:** Create a 3D grid of points by sampling `grid_size + 1` points linearly spaced between `-scale` and `scale` for X, Y, and Z axes. Stack these into a grid tensor $G$ [S1, S2, unofficial].
    $$ G = \text{meshgrid}(\text{linspace}(-scale, scale, grid\_size + 1)) $$
    Reshape $G$ into a flat list of coordinates if necessary for evaluation [S2, unofficial].

2.  **SDF Evaluation (Chunked):** Evaluate the implicit function $f$ on the grid points. To manage memory, process points in chunks of size `chunk_size` [S2, unofficial].
    For each chunk $i$ from $0$ to $N$ with step `chunk_size`:
    $$ SDF_{chunk} = implicit\_fn.get\_distance(G[i : i + chunk\_size]) $$
    Store results in a flat array $SDF_{flat}$ [S2, unofficial].

3.  **Reshape SDF:** Reshape the flat SDF array back to 3D grid dimensions:
    $$ SDF_{3D} = SDF_{flat}.view(grid\_size + 1, grid\_size + 1, grid\_size + 1) $$
    [S2, unofficial].

4.  **Marching Cubes Extraction:** Run the Marching Cubes algorithm (e.g., `skimage.measure.marching_cubes_lewiner`) on $SDF_{3D}$ with `level=thresh` and `spacing=[voxel\_size, voxel\_size, voxel\_size]` [S1, unofficial].
    *   Note: `voxel_size` is derived from `scale` and `grid_size`:
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
import torch
from skimage import measure
import plyfile

def sdf_to_ply(implicit_fn, grid_size=128, scale=0.5, thresh=0.0, chunk_size=262144, voxel_grid_origin=None):
    # Step 1: Create grid
    # Generate linspace from -scale to scale
    axis = np.linspace(-scale, scale, grid_size + 1)
    x, y, z = np.meshgrid(axis, axis, axis, indexing='ij')
    grid_points = np.stack((x, y, z), axis=-1).reshape(-1, 3)
    
    # Step 2: Evaluate SDF in chunks
    num_points = grid_points.shape[0]
    sdf_flat = np.zeros(num_points, dtype=np.float32)
    
    for start in range(0, num_points, chunk_size):
        end = min(start + chunk_size, num_points)
        chunk = grid_points[start:end]
        
        # Convert to tensor if implicit_fn expects torch tensors
        chunk_tensor = torch.FloatTensor(chunk)
        
        # Evaluate implicit function
        # [G] Assuming implicit_fn.get_distance returns a numpy array or tensor of shape (N,)
        sdf_chunk = implicit_fn.get_distance(chunk_tensor)
        
        # Handle tensor output
        if isinstance(sdf_chunk, torch.Tensor):
            sdf_chunk = sdf_chunk.cpu().numpy()
            
        sdf_flat[start:end] = sdf_chunk
        
    # Step 3: Reshape to 3D grid
    sdf_3d = sdf_flat.reshape(grid_size + 1, grid_size + 1, grid_size + 1)
    
    # Step 4: Marching Cubes
    # Calculate voxel size
    voxel_size = (2 * scale) / grid_size
    
    # Run marching cubes
    # [G] skimage.measure.marching_cubes returns (verts, faces, normals, values)
    verts, faces, normals, values = measure.marching_cubes(
        sdf_3d, 
        level=thresh, 
        spacing=(voxel_size, voxel_size, voxel_size)
    )
    
    # Step 5: Coordinate Transformation
    # If voxel_grid_origin is provided, add it
    if voxel_grid_origin is not None:
        verts += voxel_grid_origin
    else:
        # Normalize if verts are in index space (0 to grid_size)
        # Note: marching_cubes output is already scaled by spacing, 
        # but if spacing was 1.0, we might need manual scaling.
        # The source [S2, unofficial] suggests normalization: (verts/grid_size - 0.5)*2*scale
        # However, marching_cubes with spacing argument handles this.
        # We follow [S1, unofficial] which uses spacing argument, so verts are in world units.
        pass
        
    # Step 6: Construct PLY
    vertices = np.zeros(len(verts), dtype=[('x', 'f4'), ('y', 'f4'), ('z', 'f4')])
    vertices['x'] = verts[:, 0]
    vertices['y'] = verts[:, 1]
    vertices['z'] = verts[:, 2]
    
    faces = np.zeros(len(faces), dtype=[('vertex_indices', 'i4', (3,))])
    faces['vertex_indices'] = faces
    
    # Create PLY elements
    vertex_element = plyfile.PlyElement.describe(vertices, 'vertex')
    face_element = plyfile.PlyElement.describe(faces, 'face')
    
    plyfile.PlyData([vertex_element, face_element]).write('output.ply')
```

**Edge Cases and Pitfalls:**
*   **Memory Overflow:** Evaluating the entire grid at once can exceed GPU/CPU memory for large `grid_size`. Chunking is essential [S2, unofficial].
*   **Precision:** Use `float32` for SDF values and vertices to balance precision and memory, but be aware of aliasing artifacts at low resolutions [S1, S2, unofficial].
*   **Boundary Conditions:** Ensure the grid extends sufficiently beyond the object of interest to capture the zero-level set [S2, unofficial].
*   **Voxel Origin:** Incorrect handling of `voxel_grid_origin` leads to meshes displaced from the expected world coordinates [S1, unofficial].

## Subtopics

### SDF to Mesh Extraction
Extracts a mesh from a known SDF using FlexiCubes, which supports adaptive resolution and differentiable rendering [S3, unofficial]. It differs from standard Marching Cubes by using a cube-based topology that allows for smoother transitions and better handling of complex geometries [S3, unofficial]. Choose this when high-fidelity, differentiable surface extraction is required [S3, unofficial]. Link: [FlexiCubes](https://github.com/nv-tlabs/FlexiCubes).

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of grid points, dominated by the implicit function evaluation [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the SDF grid and grid points [S2, unofficial].
*   **Performance:** Depends heavily on the cost of `implicit_fn`. Chunking mitigates memory pressure but adds overhead [S2, unofficial].
*   **Strengths:** Simple to implement; leverages robust Marching Cubes implementations [S1, unofficial].
*   **Limitations:** Fixed resolution grid may not capture fine details efficiently; aliasing artifacts at low resolutions [S2, unofficial].
*   **When to Use:** When the SDF is available on a regular grid and high performance is not critical [S2, unofficial].
*   **When Not to Use:** For very large scenes or when adaptive resolution is needed [S3, unofficial].

## Background
The method originates from the need to visualize implicit surfaces defined by SDFs in computer graphics and 3D reconstruction [S1, S2, unofficial]. It builds on the classic Marching Cubes algorithm [S1, unofficial]. The specific implementation details, such as chunked evaluation and PLY output, are derived from practical implementations in neural surface reconstruction pipelines [S2, unofficial]. FlexiCubes represents a more recent development for differentiable mesh extraction [S3, unofficial].

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S2] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S3] GitHub - nv-tlabs/FlexiCubes, https://github.com/nv-tlabs/FlexiCubes, fetched 2026-10-27 (unofficial)
