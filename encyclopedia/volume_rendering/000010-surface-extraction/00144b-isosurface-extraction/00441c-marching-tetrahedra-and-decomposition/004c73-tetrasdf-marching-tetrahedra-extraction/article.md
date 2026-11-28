# TetraSDF Marching Tetrahedra Extraction

Also called: Analytic Isosurface Extraction, TetraSDF extraction, TetraSDF meshing, subpoly extraction

## Overview
TetraSDF Marching Tetrahedra Extraction extracts a triangular mesh from a trained Signed Distance Function (SDF) neural network using an analytic isosurface extraction technique on a multi-resolution tetrahedral grid [S1, unofficial]. It differs from standard grid-based marching tetrahedra by operating directly on the continuous output of a neural network rather than pre-computed discrete grid values, allowing for high-fidelity reconstruction from implicit representations [S1, unofficial]. It builds on [Marching Tetrahedra and Decomposition](11174-marching-tetrahedra-and-decomposition.md).

## How it works
**Inputs:**
- `net`: A trained SDF neural network model capable of evaluating $SDF(x)$ for any 3D point $x$.
- `DIM`: Dimensionality of the domain (typically 3).
- `CANVAS_SIZE`: The scale of the bounding box (typically 1.0) [S1, unofficial].
- `training_data.R`: A scaling factor used to map normalized coordinates to the training data's coordinate space [S1, unofficial].
- `eps_sdf`: A threshold for determining if a sample point is close enough to the isosurface (value 0) [S1, unofficial].
- `decimal`: Number of decimal places for vertex rounding to handle floating-point precision during deduplication [S1, unofficial].
- `tol_geom`: Geometric tolerance for edge connectivity/intersection thresholds [S1, unofficial].
- `chunk_size`: Batch size for processing vertices and planes to manage memory [S1, unofficial].
- `use_fp64`: Boolean flag to enable double-precision arithmetic for edge construction [S1, unofficial].

**Outputs:**
- `vertices`: Tensor of shape $[N, 3]$ containing the 3D coordinates of the mesh vertices [S1, unofficial].
- `faces`: Tensor of shape $[M, 3]$ containing indices into the vertices tensor defining the triangular faces [S1, unofficial].

**Data Structures:**
- `verts_all`: A list of tensors, each containing vertices extracted from a specific resolution level or batch [S1, unofficial].
- `planes_u`: Internal representation of the tetrahedral grid planes/samples [S1, unofficial].

**Steps:**
1. **Initialization**: Initialize `verts_all` as an empty list. Determine the output device `dev` and data type `dtype` [S1, unofficial].
2. **Iterative Extraction**: Iterate over chunks of sample points $u$ from the multi-resolution tetrahedral grid [S1, unofficial].
3. **Coordinate Transformation**: For each chunk, compute world coordinates:
   $$ x_{\text{world}} = u \times 2.0 - 1.0 $$
   This maps the normalized domain to $[-1, 1]$ [S1, unofficial].
4. **SDF Evaluation**: Query the neural network to get the SDF value for each point:
   $$ sdf\_val = \text{net.sdf}(x_{\text{world}})[:, 0].\text{abs()} $$
   The network outputs a tensor; the first column is taken [S1, unofficial].
5. **Isosurface Masking**: Identify points close to the zero-level set:
   $$ mask = sdf\_val < \epsilon_{\text{sdf}} $$
   If no points satisfy this condition, discard the current chunk and continue to the next iteration [S1, unofficial].
6. **Filtering**: Apply the mask to the sample points:
   $$ u_{\text{filtered}} = u[mask] $$
7. **Storage**: If $u_{\text{filtered}}.\text{numel()} > 0$, append $u_{\text{filtered}}.\text{to(dtype)}$ to `verts_all` [S1, unofficial].
8. **Empty Check**: If `verts_all` is empty after all iterations, return empty tensors: `vertices` shape $[0, 3]$ dtype `dtype` device `dev`, and `edges` shape $[0, 2]$ dtype `torch.int32` device `dev` [S1, unofficial].
9. **Vertex Concatenation**: Concatenate all vertex lists:
   $$ vertices = \text{torch.cat}(verts\_all, \text{dim}=0) $$
10. **Vertex Deduplication**: Deduplicate vertices to merge nearby points caused by floating-point inaccuracies or overlapping grid regions:
    $$ vertices = \text{torch.unique}(\text{torch.round}(vertices, \text{decimals}=decimal), \text{dim}=0) $$
11. **Edge Construction**: Build the edge tensor using `build_edges_tensorized` with the deduplicated `vertices`, `planes_u`, `tol_geom`, `dev`, `v_chunk=chunk_size`, `p_chunk=chunk_size`, and `use_fp64` [S1, unofficial].
12. **Edge Casting**: Cast the resulting edge tensor to `torch.long` [S1, unofficial].
13. **Vertex Scaling**: Scale the vertices back to the original training data coordinate space:
    $$ vertices = vertices / training\_data.R $$
14. **Mesh Construction**: Construct the final mesh object:
    $$ \text{trimesh.Trimesh}(vertices, faces, process=\text{False}) $$

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
    
    # The sources do not specify the algorithm for generating the multi-resolution tetrahedral grid points u [S1, unofficial].
    # u is assumed to be a pre-computed tensor of shape [N, 3] in normalized space.
    u = torch.empty(0)  # [S1, unofficial]
    
    for i in range(0, u.shape[0], chunk_size):
        u_chunk = u[i:i + chunk_size]
        x_world = u_chunk * 2.0 - 1.0
        sdf_val = net.sdf(x_world)[:, 0].abs()
        mask = sdf_val < eps_sdf
        
        if mask.any():
            u_filtered = u_chunk[mask]
            verts_all.append(u_filtered.to(dtype))
            
    if not verts_all:
        return torch.empty((0, 3), dtype=dtype, device=dev), torch.empty((0, 2), dtype=torch.int32, device=dev)
        
    vertices = torch.cat(verts_all, dim=0)
    vertices = torch.unique(torch.round(vertices, decimals=decimal), dim=0)
    
    # planes_u represents the tetrahedral grid planes/structure [S1, unofficial]
    planes_u = torch.empty(0)  # [S1, unofficial]
    
    # The internal implementation of build_edges_tensorized is not provided in the sources [S1, unofficial].
    edges = build_edges_tensorized(vertices, planes_u, tol_geom, dev, 
                                   v_chunk=chunk_size, p_chunk=chunk_size, use_fp64)
    edges = edges.to(torch.long)
    
    # faces are derived from the edge connectivity [S1, unofficial]
    faces = edges_to_faces(edges)  # [G]
    
    vertices = vertices / training_data.R
    mesh = trimesh.Trimesh(vertices, faces, process=False)
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
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Not given in the sources.
- **Strengths**: High-fidelity extraction from neural SDFs, handles complex topologies [S1, unofficial].
- **Limitations**: Requires a trained SDF network, sensitive to the choice of `eps_sdf` and `decimal` [S1, unofficial].

## Background
TetraSDF was introduced by NAVER Corp. as a method for 3D shape representation and reconstruction using neural SDFs and tetrahedral grids [S1, unofficial].

## Sources
[S1] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
