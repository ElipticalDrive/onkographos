# TetraSDF Skeleton Generation


## Overview
This method extracts the topological skeleton of a Signed Distance Function (SDF) volume by intersecting axis-aligned and diagonal planes, filtering candidates against the SDF network, and deduplicating via coordinate rounding [S1, unofficial]. It builds on [RobotSDF](13070-robotsdf.md) by operating on the SDF values produced by that parent method [S1, unofficial].

## How it works
**Inputs:**
- `sdf_net`: Optional neural network function $SDF(x)$ returning scalar SDF values for input points $x \in \mathbb{R}^3$ [S1, unofficial].
- `res_levels`: List of integers defining grid resolutions, e.g., `[2, 5, 12, 32]` [S1, unofficial].
- `use_A`: Boolean flag to apply an optional transformation matrix $A$ to normal vectors [S1, unofficial].
- `tol_geom`: Geometric tolerance for bounding box checks (default $10^{-6}$) [S1, unofficial].
- `tol_dir`: Determinant threshold to skip degenerate plane triplets (default $10^{-12}$) [S1, unofficial].
- `eps_sdf`: SDF threshold for filtering, scaled by resolution (see parameters) [S1, unofficial].

**Outputs:**
- `skeleton_points`: A tensor of shape $(N, 3)$ containing 3D coordinates of skeleton vertices [S1, unofficial].

**Data Structures:**
- `planes_u`: A dictionary mapping each normal vector $n$ to its corresponding offset values $d$, forming plane equations $n \cdot x + d = 0$ [S1, unofficial].

**Parameters:**
- `eps_sdf` scales with the maximum resolution in `res_levels` ($max\_res$):
  - $5 \times 10^{-1}$ if $max\_res < 40$
  - $1 \times 10^{-1}$ if $max\_res < 80$
  - $5 \times 10^{-2}$ otherwise [S1, unofficial].

**Steps:**
1. Define 6 normal vectors for axis-aligned and diagonal directions: $[1,0,0], [0,1,0], [0,0,1], [1,-1,0], [1,0,-1], [0,1,-1]$, normalize them, and optionally transform by matrix $A$ if `use_A` is enabled [S1, unofficial].
2. For each normal, compute offset values $d$ based on `res_levels` using helper functions `_make_axis_offsets_filtered` or `_make_diag_offsets_filtered`, which calculate grid-aligned offsets with optional center shifts and test bands [S1, unofficial].
3. Form plane equations $(n, d)$ from the dictionary and store in `planes_u` [S1, unofficial].
4. Iterate over all combinations of 3 normals (triplets) [S1, unofficial]. For each valid triplet where $\lvert \det(M) \rvert > \text{tol\_dir}$, compute inverse matrix $M^{-1}$ [S1, unofficial]. Singular triplets ($\lvert \det(M) \rvert \le \text{tol\_dir}$) are skipped to avoid numerical instability during matrix inversion [S1, unofficial].
5. Compute 3D grid of offsets via `meshgrid`, stack into $D$, and transform to $u = D M^{-T}$ [S1, unofficial].
6. Filter $u$ to keep points inside $[0, 1]^3$ using $\text{mask} = (u \ge -\text{tol\_geom}).\text{all}(\text{dim}=1) \ \& \ (u \le 1 + \text{tol\_geom}).\text{all}(\text{dim}=1)$ [S1, unofficial].
7. If a network is provided, evaluate SDF at $x_{\text{world}} = u \times 2.0 - 1.0$ and keep points where $\lvert \text{SDF} \rvert < \text{eps\_sdf}$ [S1, unofficial].
8. Collect valid $u$ points, round to decimal places, and deduplicate via `torch.unique` [S1, unofficial]. Empty plane lists or missing SDF filtering results in empty vertex sets, handled by early return of empty tensors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from itertools import combinations

def generate_skeleton_gpu(sdf_net, res_levels, use_A=False, A=None, tol_geom=1e-6, tol_dir=1e-12):
    """
    Generate skeleton points from an SDF using plane intersection method.
    
    Args:
        sdf_net: Optional neural network function SDF(x) -> scalar
        res_levels: List of integer resolutions
        use_A: Boolean flag for matrix A transformation
        A: Transformation matrix (optional)
        tol_geom: Geometric tolerance
        tol_dir: Determinant threshold
        
    Returns:
        skeleton_points: Tensor of shape (N, 3)
    """
    # Step 1: Define Normal Vectors
    normals_raw = torch.tensor([
        [1, 0, 0], [0, 1, 0], [0, 0, 1],
        [1, -1, 0], [1, 0, -1], [0, 1, -1]
    ], dtype=torch.float32)
    normals = normals_raw / normals_raw.norm(dim=1, keepdim=True)
    if use_A and A is not None:
        normals = normals @ A.T

    # Step 2 & 3: Compute Offsets and Form Plane Equations
    planes_u = {}
    for n in normals:
        offsets = []
        for res in res_levels:
            # Standard grid-aligned offsets with center shift [G]
            d_vals = torch.linspace(-0.5, 0.5, res + 1)[:-1]
            offsets.append(d_vals)
        planes_u[tuple(n.tolist())] = torch.cat(offsets)

    valid_points = []

    # Step 4: Iterate Over Triplets
    for triplet in combinations(normals, 3):
        n1, n2, n3 = triplet
        M = torch.stack([n1, n2, n3])
        if torch.abs(torch.det(M)) <= tol_dir:
            continue

        Minv = torch.inverse(M)

        # Step 5: Compute 3D Grid of Offsets
        key1, key2, key3 = tuple(n1.tolist()), tuple(n2.tolist()), tuple(n3.tolist())
        d1, d2, d3 = planes_u[key1], planes_u[key2], planes_u[key3]

        D1, D2, D3 = torch.meshgrid(d1, d2, d3, indexing='ij')
        D = torch.stack([D1, D2, D3], dim=-1).reshape(-1, 3)
        u = D @ Minv.T

        # Step 6: Filter Points
        mask = (u >= -tol_geom).all(dim=1) & (u <= 1 + tol_geom).all(dim=1)
        u_filtered = u[mask]

        if sdf_net is not None:
            x_world = u_filtered * 2.0 - 1.0
            sdf_values = sdf_net(x_world)
            max_res = max(res_levels)
            eps_sdf = 5e-1 if max_res < 40 else (1e-1 if max_res < 80 else 5e-2)
            sdf_mask = torch.abs(sdf_values) < eps_sdf
            u_filtered = u_filtered[sdf_mask]

        valid_points.append(u_filtered)

    # Step 8: Handle empty results and deduplicate
    if not valid_points:
        return torch.empty((0, 3))

    all_points = torch.cat(valid_points, dim=0)
    all_points = torch.round(all_points, decimals=6)
    unique_points, _ = torch.unique(all_points, dim=0, return_inverse=False)
    return unique_points
```

## Subtopics

### Volumetric Skeleton Animation
Compute a volumetric skeleton directly from the volume using the distance transform, deform the skeleton using a standard animation package, and reconstruct the volume about the deformed skeleton using the distance field [S2, book]. Used to create realistic animations of the Visible Human Dataset [S2, book].

## Operating characteristics
- **Time Complexity:** The SDF evaluation is the most expensive step, scaling with the number of candidate points [S1, unofficial]. Exact asymptotic bounds are Not given in the sources [S1, unofficial].
- **Memory Complexity:** Proportional to the number of candidate points generated before filtering [S1, unofficial]. Exact asymptotic bounds are Not given in the sources [S1, unofficial].
- **Strengths:** GPU-accelerated via PyTorch, efficiently samples the skeleton by leveraging geometric constraints [S1, unofficial].
- **Limitations:** Relies on the quality of the SDF network, sensitive to the choice of `res_levels` and `eps_sdf`, and skips degenerate plane triplets which may miss some skeleton parts if the geometry is complex [S1, unofficial].

## Background
TetraSDF is a method for generating signed distance functions for robot models described by a `pytorch_kinematics` Chain, conditioned on a joint configuration [S1, unofficial]. The skeleton generation method is part of the TetraSDF framework, developed by NAVER Corp., and is designed to extract topological features from the SDF volume for further processing or visualization [S1, unofficial].

## Sources
[S1] TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
[S2] Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
