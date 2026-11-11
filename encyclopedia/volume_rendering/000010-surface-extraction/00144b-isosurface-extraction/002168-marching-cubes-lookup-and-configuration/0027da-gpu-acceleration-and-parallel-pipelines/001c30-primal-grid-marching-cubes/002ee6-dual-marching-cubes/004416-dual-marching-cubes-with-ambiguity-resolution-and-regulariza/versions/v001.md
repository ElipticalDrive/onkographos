# Dual Marching Cubes with Ambiguity Resolution and Regularization

Also called: DMC with Ambiguity Resolution and Regularization, Differentiable Dual Marching Cubes, Isosurface Extraction via Dual Vertices

## Overview
This method extracts an isosurface from a voxel grid by computing dual vertices for grid cubes based on topology cases, resolving topological ambiguities (specifically cases C16 and C19) by checking adjacent cubes, and optionally optimizing vertex positions using a Quadratic Error Function (QEF) with gradient normals or linear interpolation with learned weights [S1, unofficial]. It builds on [Dual Marching Cubes](906-dual-marching-cubes.md) by adding explicit ambiguity resolution and regularization mechanisms to ensure manifold geometry and improved surface fidelity [S1, unofficial].

## How it works
**Inputs:**
- `x_nx3`: Tensor of voxel grid vertex coordinates [S1, unofficial].
- `s_n`: Scalar field values at grid vertices [S1, unofficial].
- `cube_fx8`: Indices of 8 vertices for each cube in the voxel grid [S1, unofficial].
- `res`: Resolution of the voxel grid (int or list of 3 ints) [S1, unofficial].
- `beta_fx12`: Optional weight parameters for cube edges (12 per cube) [S1, unofficial].
- `alpha_fx8`: Optional weight parameters for cube corners (8 per cube) [S1, unofficial].
- `gamma_f`: Optional weight parameters for quad splitting (1 per cube) [S1, unofficial].
- `grad_func`: Optional callable to compute surface gradients at 3D positions [S1, unofficial].

**Outputs:**
- `vertices`: Extracted mesh vertices [S1, unofficial].
- `faces`: Extracted mesh faces (triangles or tets) [S1, unofficial].
- `L_dev`: Regularizer loss (Mean Absolute Deviation) per dual vertex [S1, unofficial].

**Data Structures:**
- `surf_edges`: Tensor of shape `[N_edges, 2]` containing indices of grid vertices defining surface-intersecting edges [S1, unofficial].
- `idx_map`: Tensor of shape `[N_cubes, 12]` mapping local cube edge index to global unique surface edge index (-1 if not surface edge) [S1, unofficial].
- `case_ids`: Tensor of shape `[N_cubes]` containing topology case IDs [S1, unofficial].
- `vd_idx_map`: Tensor of shape `[N_cubes, 12]` mapping local cube edge index to global dual vertex index [S1, unofficial].
- `cube_corners_idx`: Tensor of shape `[8]` with values `[1, 2, 4, 8, 16, 32, 64, 128]` (powers of 2) [S1, unofficial].
- `cube_edges`: Tensor of shape `[24]` defining edge connections [S1, unofficial].
- `edge_dir_table`: Tensor of shape `[12]` defining edge directions [S1, unofficial].
- `dir_faces_table`: Tensor of shape `[3, 4, 2]` defining face connections [S1, unofficial].
- `adj_pairs`: Tensor of shape `[8]` defining adjacent cube pairs [S1, unofficial].

**Steps:**

1. **Identify surface-intersecting cubes:**
   Calculate the occupancy sum for each cube. A cube intersects the surface if the sum of its 8 corner occupancy flags (boolean, 1 if positive, 0 if negative) is strictly between 0 and 8.
   ```python
   occ_fx8 = (s_n[cube_fx8] > 0).long()
   occ_sum = occ_fx8.sum(dim=-1)
   surf_cubes = (occ_sum > 0) & (occ_sum < 8)
   ```

2. **Identify surface-intersecting edges:**
   For each cube, check its 12 edges. An edge intersects the surface if the signs at its two endpoints are opposite (sum of boolean occupancy flags equals 1). Assign a unique global index to each unique intersecting edge.
   ```python
   # Flatten cube edges to get all edge endpoints
   unique_edges = cube_fx8[surf_cubes][:, cube_edges].reshape(-1, 2)
   # Check if edge intersects surface
   mask_edges = occ_n[unique_edges.reshape(-1)].reshape(-1, 2).sum(-1) == 1
   # Get unique edges and their global indices
   surf_edges, inverse_indices = torch.unique(unique_edges[mask_edges], return_inverse=True, dim=0)
   # Map local cube edge index to global unique surface edge index
   idx_map = torch.full((surf_cubes.sum(), 12), -1, dtype=torch.long, device=device)
   idx_map[:, :] = inverse_indices.reshape(-1, 12)
   ```

3. **Compute topology case IDs:**
   Calculate the case ID for each surface cube by summing the product of corner occupancy bits and pre-defined corner indices (powers of 2).
   ```python
   case_ids = (occ_fx8[surf_cubes] * cube_corners_idx.to(device).unsqueeze(0)).sum(-1)
   ```

4. **Resolve topological ambiguities:**
   For cubes with ambiguous cases (indicated by `problem_config[..., 0] == 1`), check adjacent cubes. If an ambiguous cube shares a face with another ambiguous cube (specifically cases C16 and C19), invert the case ID to resolve the ambiguity.
   ```python
   problem_config = check_table[case_ids] # check_table contains ambiguity flags
   to_check = problem_config[..., 0] == 1
   problem_config = problem_config[to_check]
   
   # Construct full 3D array for all cubes
   if not isinstance(res, (list, tuple)):
       res = [res, res, res]
   problem_config_full = torch.zeros(list(res) + [5], device=device, dtype=torch.long)
   vol_idx = torch.nonzero(problem_config_full[..., 0] == 0)
   vol_idx_problem = vol_idx[surf_cubes][to_check]
   problem_config_full[vol_idx_problem[..., 0], vol_idx_problem[..., 1], vol_idx_problem[..., 2]] = problem_config
   
   # Check adjacent cubes
   vol_idx_problem_adj = vol_idx_problem + adj_pairs # Adjust for adjacency
   # Clamp to grid bounds
   vol_idx_problem_adj = torch.clamp(vol_idx_problem_adj, 0, torch.tensor(res) - 1)
   problem_config_adj = problem_config_full[vol_idx_problem_adj[..., 0], vol_idx_problem_adj[..., 1], vol_idx_problem_adj[..., 2]]
   
   # Invert case IDs if adjacent cube is also ambiguous
   to_invert = (problem_config_adj[..., 0] == 1)
   case_ids.index_put_((idx,), problem_config[to_invert][..., -1])
   ```

5. **Normalize weights:**
   Convert learnable parameters to non-negative weights. Beta (12 edges per cube) and Alpha (8 corners per cube) use `tanh` scaling: `weight * scale + 1`. Gamma (1 per cube) uses `sigmoid` scaling: `sigmoid * scale + (1-scale)/2`. If inputs are None, default to ones.
   ```python
   def normalize_weights(beta_fx12, alpha_fx8, gamma_f, weight_scale):
       if beta_fx12 is None:
           beta_fx12 = torch.ones_like(beta_fx12) # Placeholder for actual initialization
       if alpha_fx8 is None:
           alpha_fx8 = torch.ones_like(alpha_fx8)
       if gamma_f is None:
           gamma_f = torch.ones_like(gamma_f)
       
       beta_fx12 = torch.tanh(beta_fx12) * weight_scale + 1
       alpha_fx8 = torch.tanh(alpha_fx8) * weight_scale + 1
       gamma_f = torch.sigmoid(gamma_f) * weight_scale + (1 - weight_scale) / 2
       return beta_fx12, alpha_fx8, gamma_f
   ```

6. **Compute zero-crossings:**
   For each surface-intersecting edge, compute the linear interpolation point where the scalar field crosses zero.
   ```python
   # Linear interpolation: u_e = (sum(x_i * w_i)) / (sum(w_i))
   # Weights are concatenated as [w_1, -w_0] to handle signs
   surf_edges_s = s_n[surf_edges]
   surf_edges_x = x_nx3[surf_edges]
   edges_weight = torch.cat([surf_edges_s[:, 1:], -surf_edges_s[:, :1]], dim=-1)
   ue = (surf_edges_x * edges_weight.unsqueeze(-1)).sum(dim=1) / edges_weight.sum(dim=1, keepdim=True)
   ```

7. **Group edges for dual vertices:**
   For each cube, use the `dmc_table` indexed by `case_id` to determine which edges contribute to which dual vertices. Map these edge indices to global dual vertex indices (`edge_group_to_vd`).
   ```python
   # dmc_table maps case_id and local edge index to global dual vertex index
   edge_group_to_vd = dmc_table[case_ids, idx_map]
   ```

8. **Compute dual vertex positions:**
   - **Gradient/QEF mode:** If `grad_func` is provided, compute normals at zero-crossings. For each dual vertex, gather the positions of its associated zero-crossings (`verts_group_pos`) and normals. Solve a QEF: `A = norms`, `B = (positions * norms).sum()`. Add regularization: `A_reg = eye * qef_reg_scale`, `B_reg = qef_reg_scale * centroid`. Solve `lstsq(A, B)`.
     ```python
     if grad_func is not None:
         # Compute normals at zero-crossings
         normals = grad_func(ue)
         # Gather positions and normals for each dual vertex
         verts_group_pos = ue[edge_group_to_vd]
         norms = normals[edge_group_to_vd]
         
         # Solve QEF: argmin_v ||Av - B||^2
         A = norms
         B = (verts_group_pos * norms).sum(dim=1)
         
         # Regularization
         A_reg = torch.eye(3, device=device) * qef_reg_scale
         centroid = verts_group_pos.mean(dim=1)
         B_reg = qef_reg_scale * centroid
         
         A_aug = torch.cat([A, A_reg], dim=1)
         B_aug = torch.cat([B, B_reg], dim=1)
         
         dual_verts = torch.linalg.lstsq(A_aug, B_aug).solution.squeeze(-1)
     ```
   - **Weighted Linear Interpolation mode:** If no gradient, compute vertex position as weighted average of zero-crossings on incident edges. Weight is `beta`. Normalize by sum of `beta`.
     ```python
     else:
         vd = torch.zeros((edge_group_to_vd.max() + 1, 3), device=device)
         beta_group = beta_fx12[idx_map]
         ue_group = ue
         vd.index_add_(0, edge_group_to_vd, source=ue_group * beta_group.unsqueeze(-1))
         beta_sum = beta_group.sum(dim=1)
         vd = vd / beta_sum.unsqueeze(-1)
         dual_verts = vd
     ```

9. **Compute regularization loss:**
   Calculate the Mean Absolute Deviation (MAD) of the distance between the computed dual vertex and the zero-crossings on its incident edges.
   ```python
   dist = torch.norm(dual_verts[edge_group_to_vd] - ue, dim=-1)
   mean_l2 = dist.mean(dim=1)
   mad = (dist - mean_l2[edge_group_to_vd]).abs()
   L_dev = mad.mean()
   ```

## Operating characteristics
- **Time Complexity:** Linear in the number of cubes and edges, dominated by tensor operations on GPU [S1, unofficial].
- **Memory Complexity:** Linear in the number of cubes and edges, storing intermediate tensors for edges, case IDs, and dual vertices [S1, unofficial].
- **Strengths:** Resolves topological ambiguities (C16/C19) to ensure manifold geometry; supports gradient-based optimization via differentiable QEF or weighted interpolation; flexible weight parameters for surface adjustment [S1, unofficial].
- **Limitations:** Requires gradient function for QEF mode; ambiguity resolution depends on adjacent cube checks, which may be complex for large grids; regularization parameters need tuning [S1, unofficial].
- **When to use:** When extracting isosurfaces from voxel grids with potential topological ambiguities and when gradient information is available for improved surface fidelity [S1, unofficial].
- **When not to use:** For simple grids without ambiguities where standard Dual Marching Cubes suffices; when gradient computation is expensive or unavailable and weighted interpolation is insufficient [S1, unofficial].

## Background
This method is part of the FlexiCubes framework, introduced in the SIGGRAPH 2023 paper "Flexible Isosurface Extraction for Gradient-Based Mesh Optimization" [S1, unofficial]. It enhances mesh quality and geometric fidelity by adjusting the surface representation based on gradient optimization, building on the Dual Marching Cubes approach [S1, unofficial].

## Sources
[S1] RomanKotov/isosurface-extraction-comparison, https://github.com/RomanKotov/isosurface-extraction-comparison, fetched 2026-10-27 (unofficial)
