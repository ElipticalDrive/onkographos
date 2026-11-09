# Differentiable Marching Cubes

Also called: DiffMC, Flexible Isosurface Extraction for Gradient-Based Mesh Optimization

## Overview
Differentiable Marching Cubes (DiffMC) is a mesh extraction method that generates explicit, watertight manifold meshes from scalar fields (typically Signed Distance Fields) by optimizing learnable grid deformations and per-cube weights [S2]. It builds on [Neural and Learning-based Extraction](6371-neural-and-learning-based-extraction.md) by introducing a deformable grid and differentiable FlexiCubes operator, allowing the mesh geometry to be optimized via gradient descent to fit target scalar values [S1, unofficial]. Unlike static extraction methods, DiffMC dynamically adjusts surface representation through gradient-based optimization, producing significantly smoother surfaces than tet-based alternatives like DMTet when extracting from density fields [S2].

## How it works
**Inputs:**
- Scalar field values $s_n$ at grid vertices (often SDF).
- Grid resolution $res$.
- Optimization mode: 'default' (static), 'gradient' (finite difference), or 'learn' (optimizable weights/deformations) [S1, unofficial].

**Outputs:**
- Mesh vertices $V$, faces $F$, and deviation loss $L_{dev}$ [S1, unofficial].

**Data Structures:**
The method relies on lookup tables pre-loaded as PyTorch tensors [S1, unofficial]:
- `dmc_table`: `torch.long`, shape `(256, ...)`, encodes edges for dual vertices in 256 MC configurations.
- `num_vd_table`: `torch.long`, shape `(256,)`, number of dual vertices per configuration.
- `check_table`: `torch.long`, resolves ambiguity in cases C16 and C19.
- `tet_table`: `torch.long`, for isosurface tetrahedralization.
- `cube_corners`: `torch.float`, shape `(8, 3)`, unit cube corners ordered: $(0,0,0), (1,0,0), (0,1,0), (1,1,0), (0,0,1), (1,0,1), (0,1,1), (1,1,1)$ [S1, unofficial].
- `cube_corners_idx`: `torch.long`, shape `(8,)`, powers of 2 ($2^0$ to $2^7$) for case ID retrieval [S1, unofficial].
- `cube_edges`: `torch.long`, shape `(24,)`, edge pairs: `[0, 1, 1, 5, 4, 5, 0, 4, 2, 3, 3, 7, 6, 7, 2, 6, 2, 0, 3, 1, 7, 5, 6, 4]` [S1, unofficial].
- `edge_dir_table`: `torch.long`, shape `(12,)`, maps edge index to axis (0=x, 1=y, 2=z): `[0, 2, 0, 2, 0, 2, 0, 2, 1, 1, 1, 1]` [S1, unofficial].
- `dir_faces_table`: `torch.long`, shape `(3, 4, 2)`, maps shared edge axis to shared faces of adjacent cubes [S1, unofficial].
- `adj_pairs`: `torch.long`, shape `(8,)`, neighbor cube index pairs: `[0, 1, 1, 3, 3, 2, 2, 0]` [S1, unofficial].
- `quad_split_1`, `quad_split_2`, `quad_split_train`: `torch.long`, indices for quad triangulation [S1, unofficial].

**Parameters:**
- `iterations`: Optimization steps for 'learn' mode. Default: 400 [S1, unofficial].
- `learning_rate`: Adam optimizer LR for 'learn' mode. Default: 0.05 [S1, unofficial].
- `gradient_step`: Finite difference step $h$ for 'gradient' mode. Default: $10^{-9}$ [S1, unofficial].
- `qef_reg_scale`: QEF regularization scale. Default: $10^{-3}$ [S1, unofficial].
- `weight_scale`: FlexiCubes weight scale. Default: 0.99 [S1, unofficial].
- Grid resolution: Typical 256; 128 for nvdiffrec comparison; 375 for LLFF; 340/56 for Mip-NeRF 360 foreground/background [S2].

**Steps:**

1. **Grid Construction**: Generate unit vertices and cube connectivity. Vertices are centered at origin with dimension 1. Round to $10^{-5}$ precision to merge duplicates, shift by $-0.5$, and return unique vertices and cube corner indices [S1, unofficial]. Map unit vertices to world coordinates using grid bounds $[lower, upper]$:
   $$ v_{world} = lower + (v_{unit} + 0.5) \times (upper - lower) $$
   [S1, unofficial].

2. **Scalar Field Evaluation**: Evaluate the scalar field $\phi$ at vertex positions to obtain $s_n$ (SDF) [S1, unofficial].

3. **Surface Cube Identification**: Identify cubes intersecting the isosurface by checking vertex occupancy $occ_{fx8}$ against the isovalue (typically 0 for SDF) using `cube_fx8` indices [S1, unofficial]. If no surface cubes exist, return empty tensors [S1, unofficial].

4. **Case ID Calculation**: Determine the Marching Cubes configuration ID for each surface cube using `cube_corners_idx` and occupancy [S1, unofficial].

5. **Weight Normalization**: Normalize per-cube edge ($\beta$), corner ($\alpha$), and splitting ($\gamma$) weights if in 'learn' mode [S1, unofficial].

6. **Dual Vertex Computation**:
   - Identify surface edges intersecting the isosurface [S1, unofficial].
   - Compute dual vertex positions $vd$. If a gradient function `grad_func` is provided, solve a Quadratic Error Function (QEF) for positioning (non-differentiable Manifold Dual Contouring approach) [S1, unofficial]. Otherwise, use the differentiable FlexiCubes formulation [S1, unofficial].
   - Compute deviation loss $L_{dev}$ and regularizer [S1, unofficial].

7. **Triangulation**: Convert dual vertices and edges into final mesh vertices and faces using `dmc_table` and quad splitting tables [S1, unofficial].

**Pseudocode (Learn Mode):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def diffmc_learn(sdf_fn, grid_bounds, resolution, iterations=400, lr=0.05, device='cuda'):
    """
    Differentiable Marching Cubes in 'learn' mode.
    Optimizes grid deformations and FlexiCubes weights to fit SDF.
    """
    # 1. Initialize FlexiCubes operator and tables
    fc = FlexiCubes(device=device)
    
    # 2. Construct initial voxel grid
    # unit_vertices: (N, 3), cubes: (M, 8)
    unit_vertices, cubes = fc.construct_voxel_grid(resolution)
    
    # Map to world coordinates
    lower = torch.tensor(grid_bounds[0], dtype=torch.float32, device=device)
    upper = torch.tensor(grid_bounds[1], dtype=torch.float32, device=device)
    
    # Initial grid vertices
    x_nx3 = lower + (unit_vertices + 0.5) * (upper - lower)
    
    # 3. Initialize learnable parameters
    # Deformation field: same shape as x_nx3
    deform = nn.Parameter(torch.zeros_like(x_nx3))
    
    # FlexiCubes weights: beta (12), alpha (8), gamma (1) per cube
    num_cubes = cubes.shape[0]
    weight = nn.Parameter(torch.zeros(num_cubes, 21, device=device))
    # Initialize weights with scale 0.99
    weight.data.fill_(0.99)
    
    # 4. Optimizer
    optimizer = optim.Adam([deform, weight], lr=lr)
    
    # 5. Optimization Loop
    for it in range(iterations):
        optimizer.zero_grad()
        
        # Deform grid vertices using tanh-scaled deformation
        # deform_scale is the grid spacing
        deform_scale = (upper - lower) / resolution
        grid_verts = x_nx3 + (1.0 - 5e-9) * deform_scale * torch.tanh(deform)
        
        # Evaluate SDF at deformed vertices
        x, y, z = grid_verts.split(1, dim=1)
        sdf = sdf_fn(x, y, z).reshape(-1)
        
        # Extract mesh using FlexiCubes with learnable weights
        # beta: [:12], alpha: [12:20], gamma: [20]
        vertices, faces, L_dev = fc(
            grid_verts,
            sdf,
            cubes,
            resolution,
            beta_fx12=weight[:, :12],
            alpha_fx8=weight[:, 12:20],
            gamma_f=weight[:, 20],
            training=True
        )
        
        # Loss: Mean squared error between predicted SDF and target (0)
        # Target is 0 for SDF isosurface
        target = torch.zeros_like(sdf)
        sdf_loss = torch.mean((sdf - target) ** 2)
        
        # Total loss
        loss = sdf_loss + L_dev
        
        loss.backward()
        optimizer.step()
        
        # Optional: Log or save intermediate results
        if it % 100 == 0:
            print(f"Iter {it}, Loss: {loss.item()}")

    # 6. Final Extraction
    with torch.no_grad():
        grid_verts_final = x_nx3 + (1.0 - 5e-9) * deform_scale * torch.tanh(deform)
        x, y, z = grid_verts_final.split(1, dim=1)
        sdf_final = sdf_fn(x, y, z).reshape(-1)
        
        vertices, faces, _ = fc(
            grid_verts_final,
            sdf_final,
            cubes,
            resolution,
            beta_fx12=weight[:, :12],
            alpha_fx8=weight[:, 12:20],
            gamma_f=weight[:, 20],
            training=False
        )
        
    return vertices, faces
```

## Operating characteristics
- **Time Complexity**: Depends on optimization iterations. Default 400 iterations [S1, unofficial]. Each iteration involves grid deformation, SDF evaluation, and FlexiCubes extraction.
- **Memory Complexity**: Proportional to grid resolution and number of cubes. Higher resolutions require more memory for grid vertices and weights [S2].
- **Performance**: Rendering quality (PSNR) improves with resolution, plateauing around 400. At resolution 256, PSNR is 31.19; at 400, PSNR is 31.54 [S2].
- **Strengths**: Generates smoother surfaces than DMTet, mitigating artifacts from density field non-linearity [S2]. Supports multiple fitting modes (default, gradient, learn) [S1, unofficial].
- **Limitations**: Requires optimization, making it slower than static extraction. Performance drops at very high resolutions if batch size is not adjusted [S2].

## Background
DiffMC was introduced in "NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support" (2025) [S2]. It builds on the FlexiCubes method [S1, unofficial] and addresses artifacts in tet-based meshing (DMTet) when extracting from density fields [S2].

## Sources
[S1] RomanKotov/isosurface-extraction-comparison, https://github.com/RomanKotov/isosurface-extraction-comparison, fetched 2026-10-27 (unofficial)
[S2] NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support, https://doi.org/10.1109/wacv61041.2025.00081, fetched 2026-10-29
