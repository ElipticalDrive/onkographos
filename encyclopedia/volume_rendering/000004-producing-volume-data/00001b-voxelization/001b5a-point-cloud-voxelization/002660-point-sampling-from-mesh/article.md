# Point Sampling from Mesh

## Overview
Point Sampling from Mesh generates a discrete set of 3D coordinates uniformly distributed on the surface of a polygon mesh. It serves as a bridge between continuous mesh representations and discrete point-based volume data structures, often acting as an intermediate step for voxelization or neural implicit field training. Unlike rasterization-based methods that project geometry to screen space, this method operates directly in world space by sampling the mesh surface geometry [S1, unofficial]. It is distinct from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) because it generates the points from a mesh source rather than consuming an existing point cloud, and from [Interior Voxelize](995-interior-voxelize.md) because it targets the surface boundary rather than the volume interior.

## How it works
**Inputs and Outputs:**
- **Input:** A mesh defined by vertex positions $V \in \mathbb{R}^{N_v \times 3}$ and face indices $F \in \mathbb{N}^{N_f \times 3}$.
- **Output:** A set of sampled points $P \in \mathbb{R}^{N_s \times 3}$, where $N_s$ is the requested number of points.

**Data Structures:**
The implementation relies on batching to manage memory. The core data structure is a list of sampled point arrays, concatenated at the end. For GPU execution, PyTorch3D `Meshes` objects are used. For CPU execution, `trimesh.Trimesh` objects are used.

**Parameters:**
- `num_points`: Total number of points to sample. Default: 100,000,000 [S1, unofficial].
- `batch_size`: Number of points sampled per iteration to manage memory. Default: 10,000,000 [S1, unofficial].
- `device`: Execution device, either `'cpu'` or a GPU device identifier [S1, unofficial].

**Steps:**
1.  **Bounding Box Computation:** Compute the minimum and maximum coordinates of the mesh vertices to define the spatial extent. This is primarily for validation or normalization if required by downstream steps, though the sampling itself is area-weighted [S1, unofficial].
2.  **Mesh Conversion:** Convert the input vertex and face arrays into the appropriate tensor or object format for the selected device.
    -   If `device` is not `'cpu'`: Convert vertices and faces to CUDA tensors. Wrap them in a PyTorch3D `Meshes` object.
    -   If `device` is `'cpu'`: Create a `trimesh.Trimesh` object from the vertices and faces.
3.  **Batched Sampling:** Iterate in batches to accumulate points until `num_points` is reached.
    -   Calculate the current batch size: $B_{curr} = \min(B_{batch}, N_{total} - N_{accumulated})$.
    -   **GPU Path:** Call `pytorch3d.ops.sample_points_from_meshes(mesh, num_samples=B_{curr})`. This function performs uniform area-weighted sampling on the GPU. Extract the points and convert to CPU numpy arrays.
    -   **CPU Path:** Call `trimesh.sample.sample_surface(mesh, B_{curr})`. This function performs uniform area-weighted sampling on the CPU.
4.  **Aggregation:** Append the sampled points from each batch to a list.
5.  **Finalization:** Concatenate all sampled point arrays into a single array of shape $(N_{total}, 3)$. If the total sampled points exceed `num_points` due to batch rounding, truncate the array to exactly `num_points` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def sample_points_from_mesh(verts, faces, num_points=100000000, batch_size=10000000, device='cpu'):
    """
    Samples points uniformly from a mesh surface.
    
    Args:
        verts: Vertex positions, shape (N, 3).
        faces: Face indices, shape (M, 3).
        num_points: Total number of points to sample.
        batch_size: Points to sample per batch.
        device: 'cpu' or 'cuda'.
        
    Returns:
        points: Array of shape (num_points, 3).
    """
    samples_list = []
    total_sampled = 0
    
    # Determine number of batches needed
    num_batches = (num_points + batch_size - 1) // batch_size
    
    if device != 'cpu':
        # GPU Path using PyTorch3D
        # Note: PyTorch3D expects batched input (B, N, 3)
        verts_tensor = torch.tensor(verts, dtype=torch.float32, device=device).unsqueeze(0)
        faces_tensor = torch.tensor(faces, dtype=torch.int64, device=device).unsqueeze(0)
        
        from pytorch3d.structures import Meshes
        from pytorch3d.ops import sample_points_from_meshes
        
        cu_mesh = Meshes(verts=verts_tensor, faces=faces_tensor)
        
        for i in range(num_batches):
            current_batch_size = min(batch_size, num_points - total_sampled)
            if current_batch_size <= 0:
                break
                
            # Sample points
            # Returns (B, K, 3), we take the first (and only) mesh
            sampled, _ = sample_points_from_meshes(cu_mesh, num_samples=current_batch_size)
            
            # Move to CPU and numpy
            samples_list.append(sampled[0].cpu().numpy())
            total_sampled += current_batch_size
            
    else:
        # CPU Path using Trimesh
        import trimesh
        mesh = trimesh.Trimesh(vertices=verts, faces=faces)
        
        for i in range(num_batches):
            current_batch_size = min(batch_size, num_points - total_sampled)
            if current_batch_size <= 0:
                break
                
            # trimesh.sample.sample_surface returns (points, face_indices)
            sampled_points, _ = trimesh.sample.sample_surface(mesh, current_batch_size)
            samples_list.append(sampled_points)
            total_sampled += current_batch_size
            
    # Concatenate and truncate
    if not samples_list:
        return np.empty((0, 3))
        
    all_samples = np.concatenate(samples_list, axis=0)
    
    # Truncate to exact num_points if oversampled (due to batch logic or implementation details)
    if all_samples.shape[0] > num_points:
        all_samples = all_samples[:num_points]
        
    return all_samples
```

**Edge Cases and Pitfalls:**
-   **Memory Overflow:** Large `num_points` values can cause out-of-memory errors on GPU. The batching mechanism is critical. For CPU, `trimesh` handles memory internally but large batches may still be slow [S1, unofficial].
-   **Device Mismatch:** Ensure vertices and faces are on the correct device before creating the `Meshes` object. Failure to `.to(device)` will raise an error [S1, unofficial].
-   **Degenerate Meshes:** Both PyTorch3D and Trimesh may fail or produce incorrect results if the mesh has non-manifold edges, zero-area faces, or inverted normals. Pre-processing the mesh is recommended [S1, unofficial].
-   **Uniformity:** The sampling is area-weighted. If the mesh has highly uneven triangle sizes, small triangles may be under-sampled relative to their geometric contribution if not handled by the underlying library's specific algorithm (PyTorch3D and Trimesh both use area-weighted sampling) [S1, unofficial].

## Subtopics

### KD-Tree Guided Point Cloud Sampling
A variant that augments ground truth point clouds by leveraging local density information. It preprocesses points by normalizing them to a unit cube, then uses a KD-tree to compute local sigma (distance to the 50th nearest neighbor). Samples are generated by adding noise scaled by this sigma to original points. The formula for a sample is $x_{\text{sample}} = x_{\text{point}} + \sigma \cdot \mathcal{N}(0, 1)$, where $\sigma$ is the distance to the 50th nearest neighbor [S1, unofficial]. It splits large point clouds into chunks of 60 for KD-tree queries to avoid memory overflow. The output is saved as an `.npz` file containing 'sample', 'point', and 'sample_near' arrays. The number of augmentation passes is calculated as $1000000 // \text{POINT\_NUM\_GT}$ or 1 if `POINT_NUM_GT > 1000000` [S1, unofficial].

### Latent Vector to Point Cloud Generation
A pipeline step that generates a mesh from a latent vector and then samples points from it. It creates a temporary PLY mesh file using either `create_mesh` or `create_mesh_speedup` depending on the `meshcreator_type`. If mesh creation fails, it returns `None`. It then samples points from the PLY mesh using `trimesh.sample.sample_surface`. Finally, it cleans up the temporary file, either removing it or renaming it to the provided `fname` [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(N_s)$ for sampling $N_s$ points, assuming constant-time sampling per point from the underlying library. The batching adds overhead proportional to the number of batches.
-   **Memory Complexity:** $O(N_s)$ to store the output points. Intermediate memory usage is $O(B_{batch})$ due to batching.
-   **Performance:** GPU sampling via PyTorch3D is significantly faster for large $N_s$ (e.g., >10 million points) compared to CPU sampling via Trimesh [S1, unofficial].
-   **Strengths:** Simple to implement using existing libraries; provides uniform area-weighted sampling; supports large-scale point generation via batching.
-   **Limitations:** Requires the mesh to be valid (manifold, no self-intersections); GPU method requires PyTorch3D installation; CPU method may be slow for very high point counts.
-   **When to Use:** When a discrete point representation of a mesh surface is needed for neural network input, point cloud processing, or as an intermediate step for voxelization.
-   **When Not to Use:** When volume interior information is required (use [Interior Voxelize](995-interior-voxelize.md)); when rasterization-based coverage is needed for rendering (use [Rasterization-based Voxelization](908-rasterization-based-voxelization.md)).

## Background
Point sampling from meshes is a fundamental operation in computer graphics and geometric deep learning. It allows for the conversion of continuous surface representations into discrete point sets, which are easier to process with neural networks (e.g., PointNet, DGCNN) or for generating training data for implicit surface reconstruction. The specific implementation details described here are derived from unofficial repositories used in research on deep continuous functions and neural rendering [S1, S2, unofficial].

## Sources
[S1] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
