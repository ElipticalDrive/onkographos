# TetraSDF Marching Tetrahedra Extraction

Also called: Analytic Isosurface Extraction, TetraSDF extraction, TetraSDF meshing, subpoly extraction

## Overview
TetraSDF Marching Tetrahedra Extraction is a method for extracting a triangular mesh from a trained Signed Distance Function (SDF) neural network using an analytic isosurface extraction technique on a multi-resolution tetrahedral grid [S1, unofficial]. It differs from standard grid-based marching tetrahedra by operating directly on the continuous output of a neural network rather than pre-computed discrete grid values, allowing for high-fidelity reconstruction from implicit representations [S1, unofficial]. It builds on [Marching Tetrahedra and Decomposition](11174-marching-tetrahedra-and-decomposition.md).

## How it works
**Inputs:**
- `net`: A trained SDF neural network model capable of evaluating $SDF(x)$ for any 3D point $x$.
- `DIM`: Dimensionality of the domain (typically 3).
- `CANVAS_SIZE`: The scale of the bounding box (typically 1.0).
- `training_data.R`: A scaling factor used to map normalized coordinates to the training data's coordinate space.
- `eps_sdf`: A threshold for determining if a sample point is close enough to the isosurface (value 0).
- `decimal`: Number of decimal places for vertex rounding to handle floating-point precision during deduplication.
- `tol_geom`: Geometric tolerance for edge connectivity/intersection thresholds.
- `chunk_size`: Batch size for processing vertices and planes to manage memory.
- `use_fp64`: Boolean flag to enable double-precision arithmetic for edge construction.

**Outputs:**
- `vertices`: Tensor of shape $[N, 3]$ containing the 3D coordinates of the mesh vertices.
- `faces`: Tensor of shape $[M, 3]$ containing indices into the vertices tensor defining the triangular faces.

**Data Structures:**
- `verts_all`: A list of tensors, each containing vertices extracted from a specific resolution level or batch.
- `planes_u`: Internal representation of the tetrahedral grid planes/samples (implementation detail of `sp.subpoly`).

**Steps:**

1.  **Initialization**: Initialize `verts_all` as an empty list. Determine the device (CPU/GPU) and data type for the output tensors.

2.  **Iterative Extraction**: The method typically iterates through multiple resolution levels or chunks of the domain. For each iteration or chunk:
    a.  **Sample Generation**: Generate sample points $u$ within the normalized domain (typically $[0, 1]$ or $[-1, 1]$ depending on the specific grid setup, but the code snippet suggests a mapping from $u$ to world coordinates).
    b.  **Coordinate Transformation**: Convert sample points to world coordinates:
        $$ x_{world} = u \times 2.0 - 1.0 $$
        This maps the domain to $[-1, 1]$.
    c.  **SDF Evaluation**: Query the neural network to get the SDF value for each point:
        $$ sdf\_val = net.sdf(x_{world})[:, 0] $$
        The network outputs a tensor; we take the first column.
    d.  **Isosurface Masking**: Identify points close to the zero-level set:
        $$ mask = |sdf\_val| < \epsilon_{sdf} $$
        If no points satisfy this condition, discard the current batch and continue to the next iteration.
    e.  **Filtering**: Apply the mask to the sample points:
        $$ u_{filtered} = u[mask] $$
    f.  **Storage**: If $u_{filtered}$ is not empty, append it (converted to the target dtype) to `verts_all`.

3.  **Empty Check**: If `verts_all` is empty after all iterations, return empty tensors:
    -   `vertices`: shape $[0, 3]$, dtype `dtype`, device `dev`.
    -   `edges`: shape $[0, 2]$, dtype `torch.int32`, device `dev`.

4.  **Vertex Concatenation and Deduplication**:
    a.  Concatenate all vertex lists:
        $$ vertices = \text{torch.cat}(verts\_all, dim=0) $$
    b.  Deduplicate vertices to merge nearby points caused by floating-point inaccuracies or overlapping grid regions:
        $$ vertices = \text{torch.unique}(\text{torch.round}(vertices, decimals=decimal), dim=0) $$

5.  **Edge and Face Construction**:
    a.  Call the internal function `build_edges_tensorized` with the deduplicated `vertices`, `planes_u` (representing the tetrahedral structure), `tol_geom`, `dev`, `v_chunk=chunk_size`, `p_chunk=chunk_size`, and `use_fp64`.
    b.  This function determines the connectivity between vertices based on the analytic intersection of the SDF with the tetrahedral cells.
    c.  Cast the resulting edge tensor to `torch.long`.
    d.  The function `sp.subpoly` (or equivalent internal logic) returns `polygons` (faces), `vertices`, and `faces_with_indices`.

6.  **Final Mesh Construction**:
    a.  Scale the vertices back to the original training data coordinate space if necessary (though the code snippet shows scaling *by* `1/R` in the example usage, implying the network might be trained in a normalized space or the output needs adjustment). The snippet says: `Scale vertices by 1 / training_data.R`.
    b.  Construct the final mesh object:
        ```python
        mesh = trimesh.Trimesh(vertices, triangles, process=False)
        ```

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import trimesh

def extract_mesh_tetrasdf(net, training_data, DIM=3, CANVAS_SIZE=1.0, 
                          eps_sdf=1e-4, decimal=5, tol_geom=1e-5, 
                          chunk_size=32768, use_fp64=False):
    """
    Extracts a triangular mesh from a trained SDF network using analytic isosurface extraction.
    """
    verts_all = []
    dev = next(net.parameters()).device
    dtype = torch.float32
    
    # Note: The exact sampling strategy (grid vs random) and iteration logic 
    # for 'planes_u' is encapsulated in sp.subpoly. We simulate the core logic 
    # described in the notes.
    
    # Placeholder for the multi-resolution grid generation logic
    # In the actual implementation, sp.subpoly handles the tetrahedral grid traversal.
    # Here we describe the per-point processing logic.
    
    # Simulate getting sample points u from the grid structure
    # u shape: [N, 3] in normalized space [0, 1] or similar
    # This part depends on the internal state of sp.subpoly
    
    # For the sake of a self-contained implementation based on the notes:
    # We assume sp.subpoly returns the necessary components.
    # However, to be implementable from text, we must define the steps 
    # that sp.subpoly performs internally as described.
    
    # Step 1: Generate samples and filter
    # The notes describe a loop over sample points. 
    # We will assume a function get_samples() provides the initial grid points.
    
    # Since sp.subpoly is the entry point, we detail its internal logic:
    
    polygons, vertices, faces_with_indices = sp.subpoly(net, DIM, CANVAS_SIZE, force=False)
    
    # If sp.subpoly is a black box, we cannot implement it. 
    # The notes provide the internal steps of sp.subpoly.
    # We rewrite sp.subpoly logic here:
    
    def subpoly_impl(net, DIM, CANVAS_SIZE, eps_sdf, decimal, tol_geom, chunk_size, use_fp64, training_data):
        verts_all = []
        dev = next(net.parameters()).device
        
        # 1. Generate initial sample points u. 
        # The notes imply a multi-resolution grid. 
        # We assume a function that generates points u in the domain.
        # For a complete implementation, we need the grid generation.
        # The notes do not specify the grid generation algorithm, only the filtering.
        # We will assume a standard grid or the specific TetraSDF grid generation.
        # Since the notes say "on a multi-resolution tetrahedral grid", 
        # we assume the grid is predefined or generated.
        
        # Let's assume we have a list of planes/points to process.
        # The notes mention 'planes_u'.
        
        # We will implement the filtering and deduplication logic as described.
        
        # Placeholder for grid generation
        # u = generate_tetrahedral_grid(...) 
        
        # The notes describe processing 'u' in chunks or iterations.
        # We simulate the loop.
        
        # For the purpose of this article, we assume the grid points 'u' are available.
        # In a real implementation, this would be the TetraSDF grid.
        
        # Step 2: Filter points
        # We need to process points in batches if they are large.
        # The notes describe a loop over 'u'.
        
        # Let's assume 'u' is a tensor of shape [N, 3]
        # We need to know N. The notes don't specify.
        # We will assume the grid generation is handled externally or is part of the method.
        
        # To make this implementable, we must define the grid.
        # The notes do not provide the grid generation code.
        # We will leave a comment.
        
        # u = ... # Generated from multi-resolution tetrahedral grid
        
        # Process points
        # The notes describe:
        # for each sample point u:
        #   x_world = u * 2.0 - 1.0
        #   sdf_val = net.sdf(x_world)[:, 0].abs()
        #   mask = sdf_val < eps_sdf
        #   if mask.any():
        #       u_filtered = u[mask]
        #       verts_all.append(u_filtered)
        
        # This implies a point-wise or batch-wise processing.
        
        # We will implement the batch-wise version as suggested by chunk_size.
        
        # Assume u is generated.
        # u = generate_samples(...) 
        
        # For the sake of the article, we assume the grid generation is a prerequisite.
        # The core method is the analytic extraction via SDF evaluation and filtering.
        
        # We will return the result of the filtering and deduplication.
        
        # Re-writing the logic from the notes:
        
        # 1. Get all sample points u from the tetrahedral grid.
        # This is not provided in the notes. We assume it exists.
        # u = get_tetrahedral_grid_points(...)
        
        # 2. Filter
        # We process in chunks to manage memory.
        for i in range(0, u.shape[0], chunk_size):
            u_chunk = u[i:i+chunk_size]
            x_world = u_chunk * 2.0 - 1.0
            sdf_val = net.sdf(x_world)[:, 0].abs()
            mask = sdf_val < eps_sdf
            
            if mask.any():
                u_filtered = u_chunk[mask]
                verts_all.append(u_filtered.to(dtype))
        
        # 3. Concatenate and Deduplicate
        if not verts_all:
            return torch.empty((0, 3), dtype=dtype, device=dev), torch.empty((0, 2), dtype=torch.int32, device=dev)
            
        vertices = torch.cat(verts_all, dim=0)
        vertices = torch.unique(torch.round(vertices, decimals=decimal), dim=0)
        
        # 4. Build Edges/Faces
        # This requires the tetrahedral connectivity.
        # The notes mention build_edges_tensorized.
        # We assume this function takes vertices and the grid structure (planes_u).
        # planes_u is not defined in the notes.
        # We assume it is derived from the grid generation.
        
        # edges = build_edges_tensorized(vertices, planes_u, tol_geom, dev, chunk_size, chunk_size, use_fp64)
        # faces = ... # Derived from edges and grid topology
        
        # The notes say sp.subpoly returns polygons, vertices, faces_with_indices.
        # We will assume the internal logic of sp.subpoly handles the grid connectivity.
        
        # Since we cannot implement the grid connectivity without the grid definition,
        # we will note that the grid generation is a prerequisite.
        
        # However, the notes say "Extracts a triangular mesh ... using an analytic isosurface extraction method".
        # The "analytic" part likely refers to the precise intersection calculation.
        
        # We will return the vertices and assume the faces are generated by the grid logic.
        
        # To be complete, we must define the grid.
        # The notes do not provide it.
        # We will leave a comment.
        
        return vertices, faces # Placeholder
        
    # The actual implementation relies on sp.subpoly which encapsulates the grid and edge building.
    # The notes provide the high-level steps.
    
    # We will return the result of sp.subpoly as described.
    polygons, vertices, faces_with_indices = sp.subpoly(net, DIM, CANVAS_SIZE, force=False)
    
    # Scale vertices
    vertices = vertices / training_data.R
    
    # Construct mesh
    mesh = trimesh.Trimesh(vertices, faces_with_indices, process=False)
    
    return mesh
```

**Parameters:**
- `CANVAS_SIZE`: 1.0 [S1, unofficial].
- `decimal`: Number of decimal places for rounding vertices before deduplication [S1, unofficial].
- `tol_geom`: Geometric tolerance for edge connectivity [S1, unofficial].
- `chunk_size`: 32768 [S1, unofficial].
- `use_fp64`: Boolean flag for double-precision arithmetic [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Mesh**: If no points satisfy the SDF threshold, return empty tensors [S1, unofficial].
- **Floating-Point Precision**: Use rounding and deduplication to merge vertices that are very close but not identical due to precision errors [S1, unofficial].
- **Memory Management**: Use chunking for large grids to avoid out-of-memory errors [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the number of sample points and the cost of SDF evaluation. SDF evaluation is typically $O(N)$ where $N$ is the number of samples.
- **Memory Complexity**: $O(N)$ for storing sample points and vertices.
- **Performance**: The method is efficient for high-resolution meshes due to the multi-resolution grid and chunked processing [S1, unofficial].
- **Strengths**: High-fidelity extraction from neural SDFs, handles complex topologies [S1, unofficial].
- **Limitations**: Requires a trained SDF network, sensitive to the choice of `eps_sdf` and `decimal` [S1, unofficial].

## Background
TetraSDF was introduced by NAVER Corp. as a method for 3D shape representation and reconstruction using neural SDFs and tetrahedral grids [S1, unofficial]. It aims to provide a more accurate and efficient alternative to voxel-based or standard grid-based methods.

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
