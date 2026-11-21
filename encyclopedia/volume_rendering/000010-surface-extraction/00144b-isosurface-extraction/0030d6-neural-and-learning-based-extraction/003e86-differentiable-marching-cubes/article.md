# Differentiable Marching Cubes

Also called: DiffMC, Flexible Isosurface Extraction for Gradient-Based Mesh Optimization

## Overview
Differentiable Marching Cubes (DiffMC) is a differentiable isosurface extraction method that extracts meshes from scalar fields, typically Signed Distance Fields (SDFs), by dynamically adjusting surface representation through gradient-based optimization of per-cube weight parameters and a deformable grid [S1, unofficial][S2]. It builds on [Isosurface extraction](30-isosurface-extraction.md) and distinguishes itself from siblings like [MISE](977-mise-meshing-isosurfaces-by-extracting.md) and [FlexiCubes](3669-flexicubes.md) by supporting multiple fitting modes (`learn` and `gradient`) and explicitly optimizing grid geometry and weights to fit target scalar values [S1, unofficial][S2]. Unlike [DMTet](3669-flexicubes.md) (often grouped with Tet-based methods), DiffMC generates significantly smoother surfaces and mitigates zigzag artifacts when extracting from density fields [S2].

## How it works
**Inputs:**
*   A scalar field function $\phi(\mathbf{x})$ (e.g., SDF or density) [S1, unofficial][S2].
*   Grid resolution `cells` [S1, unofficial][S2].
*   Optimization mode: `learn` (gradient-based optimization of weights and deformations) or `gradient` (finite-difference gradient approximation) [S1, unofficial].
*   Optional: Target SDF values for the `learn` mode [S1, unofficial].

**Outputs:**
*   Mesh vertices $\mathbf{V}$ and faces $\mathbf{F}$ [S1, unofficial][S2].
*   Deviation loss $L_{dev}$ (regularizer) [S1, unofficial].

**Data Structures:**
The method relies on lookup tables and tensors defined as follows [S1, unofficial]:

*   `dmc_table`: `torch.long`, shape $(256, \dots)$, encodes edges associated with each dual vertex in 256 MC configurations.
*   `num_vd_table`: `torch.long`, shape $(256,)$, holds the number of dual vertices per MC configuration.
*   `check_table`: `torch.long`, resolves ambiguity in cases C16 and C19 of DMC configurations.
*   `tet_table`: `torch.long`, lookup table for isosurface tetrahedralization.
*   `quad_split_1`: `torch.long`, indices `[0, 1, 2, 0, 2, 3]` for splitting a quad along one diagonal.
*   `quad_split_2`: `torch.long`, indices `[0, 1, 3, 3, 1, 2]` for splitting a quad along the other diagonal.
*   `quad_split_train`: `torch.long`, indices `[0, 1, 1, 2, 2, 3, 3, 0]` for splitting a quad into four triangles during training.
*   `cube_corners`: `torch.float`, shape $(8, 3)$, unit cube corners: $(0,0,0), (1,0,0), (0,1,0), (1,1,0), (0,0,1), (1,0,1), (0,1,1), (1,1,1)$.
*   `cube_corners_idx`: `torch.long`, shape $(8,)$, values $2^0 \dots 2^7$, used to retrieve case ID from vertex occupancy.
*   `cube_edges`: `torch.long`, shape $(24,)$, edge connections in pairs: `[0, 1, 1, 5, 4, 5, 0, 4, 2, 3, 3, 7, 6, 7, 2, 6, 2, 0, 3, 1, 7, 5, 6, 4]`.
*   `edge_dir_table`: `torch.long`, shape $(12,)$, maps edge indices to axis (0=x, 1=y, 2=z): `[0, 2, 0, 2, 0, 2, 0, 2, 1, 1, 1, 1]`.
*   `dir_faces_table`: `torch.long`, shape $(3, 4, 2)$, maps shared edge axes to shared faces of adjacent cubes.
*   `adj_pairs`: `torch.long`, shape $(8,)$, index pairs for neighboring cubes sharing an edge: `[0, 1, 1, 3, 3, 2, 2, 0]`.

**Parameters:**
*   `iterations`: Number of optimization steps for `learn` mode. Default: 400 [S1, unofficial].
*   `learning_rate`: Initial Adam optimizer learning rate for `learn` mode. Default: 0.05 [S1, unofficial].
*   `gradient_step`: Step size $h$ for finite difference in `gradient` mode. Default: $10^{-9}$ [S1, unofficial].
*   `qef_reg_scale`: Regularization scaling factor for QEF singularity prevention. Default: $10^{-3}$ [S1, unofficial].
*   `weight_scale`: Scale of FlexiCubes weights. Must be in $[0, 1]$. Default: 0.99 [S1, unofficial].
*   `grid_resolution`: Typical values are 256 for general results, 128 for comparison with nvdiffrec, 375 for LLFF, and 340/56 for foreground/background in Mip-NeRF 360 [S2].

**Algorithm Steps:**

1.  **Initialization:** Load DMC tables into PyTorch tensors on the target device (default 'cuda'). Initialize `qef_reg_scale` to $10^{-3}$ and `weight_scale` to 0.99 [S1, unofficial].
2.  **Voxel Grid Construction:**
    *   Generate unit vertices and cube connectivity indices using the FlexiCubes constructor for resolution `cells` [S1, unofficial].
    *   Vertices are centered at the origin with dimension length 1. Round vertices to $10^{-5}$ precision to merge duplicates.
    *   Return unique vertices shifted by $-0.5$ and cube corner indices [S1, unofficial].
    *   Map unit vertices to world coordinates using grid lower/upper bounds: $\mathbf{x}_{world} = \mathbf{x}_{unit} \times (\mathbf{upper} - \mathbf{lower}) + \mathbf{lower}$ [S1, unofficial].
3.  **Scalar Field Evaluation:** Evaluate $\phi$ at the grid vertex positions to obtain the SDF values $\mathbf{s}_n$ [S1, unofficial][S2].
4.  **Surface Cube Identification:** Identify cubes intersecting the isosurface based on $\mathbf{s}_n$ and cube vertex indices $\mathbf{cube}_{fx8}$ [S1, unofficial].
5.  **Empty Check:** If no surface cubes are found ($\sum \text{surf\_cubes} == 0$), return empty tensors for vertices, faces, and regularizer [S1, unofficial].
6.  **Weight Normalization:** Normalize per-cube edge ($\beta$), corner ($\alpha$), and splitting ($\gamma$) weights [S1, unofficial].
7.  **Case ID Calculation:** Determine the Marching Cubes configuration ID for each surface cube based on vertex occupancy [S1, unofficial].
8.  **Surface Edge Identification:** Identify edges intersecting the isosurface, returning surface edges, index mapping, edge counts, and mask [S1, unofficial].
9.  **Dual Vertex Computation:**
    *   If `grad_func` is provided (non-differentiable mode): Solve a Quadratic Error Function (QEF) for dual vertex positioning, similar to Manifold Dual Contouring, using a smart splitting strategy [S1, unofficial].
    *   If `grad_func` is not provided (differentiable mode): Use the original differentiable FlexiCubes algorithm [S1, unofficial].
    *   Compute dual vertex positions $\mathbf{v}_d$, regularizer $L_{dev}$, gamma values $\mathbf{v}_d\_\gamma$, and index mapping $\mathbf{v}_d\_\text{idx\_map}$ [S1, unofficial].
10. **Triangulation:** Convert dual vertices and edges into final mesh vertices and faces using the quad split tables [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def diffmc_extract(phi_func, cells, mode='learn', iterations=400, learning_rate=0.05, gradient_step=1e-9):
    device = 'cuda' if torch.cuda.is_available() else 'cpu'
    # 1. Initialization: Load DMC tables into PyTorch tensors on device [S1, unofficial].
    qef_reg_scale = 1e-3
    weight_scale = 0.99
    
    # 2. Voxel Grid Construction
    unit_vertices, cubes = construct_voxel_grid(cells)
    unit_vertices = torch.round(unit_vertices * 1e5) / 1e5
    unit_vertices = torch.unique(unit_vertices - 0.5, dim=0)
    lower = torch.tensor([-0.5, -0.5, -0.5], dtype=torch.float32, device=device)
    upper = torch.tensor([0.5, 0.5, 0.5], dtype=torch.float32, device=device)
    x_nx3 = lower + (unit_vertices + 0.5) * (upper - lower)
    
    # 3. Scalar Field Evaluation
    sdf = phi_func(x_nx3)
    
    # 4. Surface Cube Identification
    surf_cubes = identify_surf_cubes(sdf, cubes)
    if surf_cubes.sum() == 0:
        return torch.empty((0, 3), device=device), torch.empty((0, 3), dtype=torch.long, device=device)
    
    # 5. Weight Normalization & Case ID
    if mode == 'learn':
        num_cubes = surf_cubes.sum()
        beta_fx12 = nn.Parameter(torch.ones(num_cubes, 12, device=device) * weight_scale)
        alpha_fx8 = nn.Parameter(torch.ones(num_cubes, 8, device=device) * weight_scale)
        gamma_f = nn.Parameter(torch.ones(num_cubes, device=device) * weight_scale)
        optimizer = optim.Adam([beta_fx12, alpha_fx8, gamma_f], lr=learning_rate)
    
    # 6. Optimization Loop
    if mode == 'learn':
        for it in range(iterations):
            optimizer.zero_grad()
            deform_scale = (upper - lower) / cells
            grid_verts = x_nx3 + (1.0 - 5e-9) * deform_scale * torch.tanh(deform)
            vertices, faces, L_dev = flexicubes_operator(grid_verts, sdf, cubes, cells, beta_fx12=beta_fx12, alpha_fx8=alpha_fx8, gamma_f=gamma_f, training=True)
            target_sdf = phi_func(grid_verts)
            sdf_loss = ((target_sdf.nan_to_num(0.1) - sdf.nan_to_num(0.1))**2).mean()
            loss = sdf_loss + L_dev
            loss.backward()
            optimizer.step()
    else:
        def grad_f(x3):
            h = gradient_step
            x, y, z = x3.split(1, dim=1)
            dh = 2 * h
            df_dx = (phi_func(x + h, y, z) - phi_func(x - h, y, z)) / dh
            df_dy = (phi_func(x, y + h, z) - phi_func(x, y - h, z)) / dh
            df_dz = (phi_func(x, y, z + h) - phi_func(x, y, z - h)) / dh
            return torch.cat([df_dx, df_dy, df_dz], axis=1)
        vertices, faces, L_dev = flexicubes_operator(x_nx3, sdf, cubes, cells, grad_func=grad_f)
        
    if mode == 'learn':
        with torch.no_grad():
            vertices, faces, L_dev = flexicubes_operator(grid_verts, sdf, cubes, cells, beta_fx12=beta_fx12, alpha_fx8=alpha_fx8, gamma_f=gamma_f, training=False)
    return vertices, faces

def construct_voxel_grid(cells):
    coords = torch.linspace(-0.5, 0.5, cells + 1)
    X, Y, Z = torch.meshgrid(coords, coords, coords, indexing='ij')
    unit_vertices = torch.stack([X, Y, Z], dim=-1).reshape(-1, 3)
    idx = torch.arange(cells + 1, dtype=torch.long)
    I, J, K = torch.meshgrid(idx[:-1], idx[:-1], idx[:-1], indexing='ij')
    offsets = torch.tensor([[0,0,0], [1,0,0], [0,1,0], [1,1,0], [0,0,1], [1,0,1], [0,1,1], [1,1,1]], dtype=torch.long)
    cubes = (I.unsqueeze(-1) + offsets[:, 0]).long() * (cells+1)**2 + (J.unsqueeze(-1) + offsets[:, 1]).long() * (cells+1) + (K.unsqueeze(-1) + offsets[:, 2]).long()
    return unit_vertices, cubes

def identify_surf_cubes(sdf, cubes):
    sdf_corners = sdf[cubes]
    return (sdf_corners > 0).any(dim=1) & (sdf_corners < 0).any(dim=1)

def flexicubes_operator(grid_verts, sdf, cubes, cells, beta_fx12=None, alpha_fx8=None, gamma_f=None, training=False, grad_func=None):
    surf_cubes = identify_surf_cubes(sdf, cubes)
    if surf_cubes.sum() == 0:
        return torch.empty((0, 3), device=grid_verts.device), torch.empty((0, 3), dtype=torch.long, device=grid_verts.device), torch.tensor(0.0, device=grid_verts.device)
    if beta_fx12 is not None:
        beta_fx12 = normalize_weights(beta_fx12, alpha_fx8, gamma_f, surf_cubes)
    sdf_corners = sdf[cubes[surf_cubes]]
    occ = (sdf_corners > 0).long()
    cube_corners_idx = torch.tensor([1, 2, 4, 8, 16, 32, 64, 128], dtype=torch.long, device=sdf.device)
    case_ids = (occ * cube_corners_idx).sum(dim=1)
    surf_edges, idx_map, edge_counts, surf_edges_mask = identify_surf_edges(sdf, cubes, surf_cubes)
    # If grad_func is provided, solve QEF for dual vertex positioning (non-differentiable) [S1, unofficial]
    # Otherwise, use original differentiable FlexiCubes algorithm [S1, unofficial]
    # Compute dual vertex positions vd, regularizer L_dev, gamma values vd_gamma, and index mapping vd_idx_map
    if grad_func is not None:
        vd, L_dev, vd_gamma, vd_idx_map = compute_vd_qef(grid_verts, cubes[surf_cubes], surf_edges, sdf, case_ids, beta_fx12, alpha_fx8, gamma_f, idx_map, grad_func)
    else:
        vd, L_dev, vd_gamma, vd_idx_map = compute_vd_flexicubes(grid_verts, cubes[surf_cubes], surf_edges, sdf, case_ids, beta_fx12, alpha_fx8, gamma_f, idx_map)
    vertices, faces = triangulate(vd, surf_edges, case_ids, vd_idx_map)
    return vertices, faces, L_dev

def identify_surf_edges(sdf, cubes, surf_cubes):
    sdf_corners = sdf[cubes[surf_cubes]]
    edge_verts = cubes[surf_cubes][:, cube_edges]
    edge_sdf = sdf_corners[edge_verts]
    edge_intersects = (edge_sdf[:, :, 0] * edge_sdf[:, :, 1] < 0)
    return edge_intersects.nonzero(as_tuple=False), None, None, None

# Sources do not give implementation details for these steps [S1, unofficial]
def normalize_weights(beta_fx12, alpha_fx8, gamma_f, surf_cubes): pass
def compute_vd_qef(grid_verts, cube_verts, surf_edges, sdf, case_ids, beta_fx12, alpha_fx8, gamma_f, idx_map, grad_func): pass
def compute_vd_flexicubes(grid_verts, cube_verts, surf_edges, sdf, case_ids, beta_fx12, alpha_fx8, gamma_f, idx_map): pass
def triangulate(vd, surf_edges, case_ids, vd_idx_map): pass
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Rendering quality (PSNR) improves with DiffMC resolution, plateauing around 400. At resolution 256, PSNR is 31.19; at 400, PSNR is 31.54 [S2].
*   **Strengths:** Generates smoother surfaces than DMTet, mitigating artifacts from density field non-linearity [S2]. Supports multiple fitting modes (`learn`, `gradient`) [S1, unofficial].
*   **Limitations:** The `learn` mode requires careful tuning of learning rate and iterations. The `gradient` mode is non-differentiable with respect to the scalar field, limiting its use in end-to-end differentiable pipelines that require gradients through the meshing step [S1, unofficial].
*   **When to use:** When smooth mesh extraction from SDFs or density fields is required, especially in neural rendering pipelines [S2].
*   **When not to use:** When differentiability through the meshing step is strictly required and the `gradient` mode's non-differentiability is a blocker (use `learn` mode with care, or other differentiable methods). When computational resources are limited and high resolution is not needed (lower resolutions may introduce artifacts).

## Background
DiffMC was introduced in the paper "NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support" [S2]. It builds on the FlexiCubes algorithm [S1, unofficial] and addresses the limitations of DMTet in extracting meshes from density fields. The method is designed to produce watertight manifolds suitable for efficient and high-quality rendering [S2].

## Sources
[S1] RomanKotov/isosurface-extraction-comparison, https://github.com/RomanKotov/isosurface-extraction-comparison, fetched 2026-10-27 (unofficial)
[S2] NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support, https://doi.org/10.1109/wacv61041.2025.00081, fetched 2026-10-29
