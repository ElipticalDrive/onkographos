# Backward Differentiable Monte Carlo Voxelization


## Overview
Computes backward-mode gradients of voxel occupancy with respect to mesh vertex positions using Monte Carlo sampling [S1, unofficial]. It differs from [Forward-Mode Derivative of Monte Carlo 2D Voxelization](1876-forward-mode-derivative-of-monte-carlo-2d-voxelization.md) by propagating gradients from the output occupancy grid back to the input vertices, enabling geometry optimization from voxel supervision [S1, unofficial].

## How it works
**Inputs:**
- `vertices`: Array of shape $(N, 3)$ containing vertex positions, type `Float`.
- `faces`: Array of shape $(M, 3)$ containing vertex indices for each triangle, type `uint32_t`.
- `d_occupancy`: Input array of shape $(H, W)$ for 2D or $(D, H, W)$ for 3D, containing occupancy gradients, type `Float`.
- `height`, `width`, `depth`: Grid dimensions.

**Outputs:**
- `d_vertices`: Output array of shape $(N, 3)$ containing vertex gradients, type `Float`. Initialized to zero.

**Data Structures:**
- Temporary vectors for polygon splitting in barycentric coordinates: `remaining_x`, `poly_x`, `temp_below`, `temp_above`, `remaining_y`, `poly_xy`, `remaining_z`, `poly_bary`, `poly_xyz`.
- `dvn_vec`: Vector of shape $(3)$ storing the derivative projected onto the normal direction.

**Steps:**
1. Validate inputs using `validate_common_differential_voxelize_arguments` [S1, unofficial].
2. Dispatch to 2D or 3D backward-mode implementation with `DifferentiationMode::Backward` [S1, unofficial].
3. For each face $f$ with vertices $v_0, v_1, v_2$:
   a. Compute edge vectors: $e_1 = v_1 - v_0$, $e_2 = v_2 - v_0$.
   b. Compute normal: $n = e_1 \times e_2$.
   c. Compute normal length: $L = \|n\|$.
   d. If $L < 10^{-12}$, skip face [S1, unofficial].
   e. Compute unit normal: $\hat{n} = n / L$.
   f. For Backward mode, set $dvn\_vec = \hat{n}$ [S1, unofficial].
   g. Perform Monte Carlo integration to compute gradients. The gradient contribution for each vertex is derived from the derivative of the occupancy function with respect to vertex positions, projected onto the normal direction [S1, unofficial].
   h. Accumulate gradients into `d_vertices` for $v_0, v_1, v_2$.

**Parameters:**
- `d_vertices`: Output array of shape $(n, dim)$ containing vertex gradients, type `Float` [S1, unofficial].
- `d_occupancy`: Input array of shape $(h, w)$ or $(d, h, w)$ containing occupancy gradients, type `Float` [S1, unofficial].

**Edge Cases:**
- Zero-area triangles ($L < 10^{-12}$) are skipped [S1, unofficial].
- Precision issues near boundaries may require careful handling of Monte Carlo samples.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward_differentiable_mc_voxelize(vertices, faces, d_occupancy, height, width, depth):
    # Initialize output gradients
    d_vertices = zeros_like(vertices)
    
    # Validate inputs
    validate_common_differential_voxelize_arguments(vertices, faces, d_occupancy)
    
    # Determine dimensionality
    if d_occupancy.ndim == 2:
        dim = 2
    else:
        dim = 3
        
    # Dispatch to specific implementation
    if dim == 2:
        backward_2d(vertices, faces, d_occupancy, height, width, d_vertices)
    else:
        backward_3d(vertices, faces, d_occupancy, height, width, depth, d_vertices)
        
    return d_vertices

def backward_2d(vertices, faces, d_occupancy, height, width, d_vertices):
    for face_idx in range(len(faces)):
        i0, i1, i2 = faces[face_idx]
        v0, v1, v2 = vertices[i0], vertices[i1], vertices[i2]
        
        e1 = v1 - v0
        e2 = v2 - v0
        
        normal = cross(e1, e2)
        normal_len = norm(normal)
        
        if normal_len < 1e-12:
            continue
            
        unit_normal = normal / normal_len
        
        # Compute gradient contributions
        # The gradient is proportional to the unit normal and the occupancy gradient
        # This is a simplified representation; actual implementation uses Monte Carlo sampling
        grad_contribution = unit_normal * d_occupancy.mean() # Placeholder for MC integration
        
        d_vertices[i0] += grad_contribution
        d_vertices[i1] += grad_contribution
        d_vertices[i2] += grad_contribution

def backward_3d(vertices, faces, d_occupancy, height, width, depth, d_vertices):
    # Similar to 2D but with 3D grid sampling
    pass
```

## Subtopics

### [Differentiable Rasterization via Ray Casting](17129-differentiable-rasterization-via-ray-casting.md)
This method converts a triangle mesh into a 3D voxel volume by casting random rays from voxel centers to the top plane of the bounding box, enabling gradient-based optimization of mesh geometry from voxel supervision. It differs from standard rasterization by providing a backward pass that computes gradients with respect to vertex positions, allowing the mesh to be refined to match a target voxel occupancy. Unlike [Backward Differentiable Monte Carlo Voxelization](1997-backward-differentiable-monte-carlo-voxelization.md), which relies on Monte Carlo sampling for the backward pass, this method uses a deterministic ray-casting forward pass with a specialized geometric gradient approximation for the backward pass.

## Operating characteristics
- **Time Complexity:** Depends on the number of faces and Monte Carlo samples. Typically $O(N \cdot S)$ where $N$ is the number of faces and $S$ is the number of samples.
- **Memory Complexity:** $O(N)$ for storing vertex gradients and temporary polygon data.
- **Strengths:** Differentiable, allowing for gradient-based optimization of mesh geometry.
- **Limitations:** Stochastic nature may introduce noise in gradients. Performance depends on the number of samples required for convergence.
- **When to use:** When gradient-based optimization of mesh geometry from voxel supervision is required.
- **When not to use:** When deterministic results are needed or when computational resources are limited.

## Background
Originated from the work on differentiable voxelization of surface representations [S1, unofficial]. The method leverages Monte Carlo integration to compute gradients, providing a flexible approach to differentiable rendering and geometry optimization.

## Sources
[S1] Differentiable Voxelization of Surface Representations, https://github.com/mworchel/differentiable-voxelization, fetched 2026-10-29 (unofficial)
