# Trilinear Gradient Interpolation

Also called: Trilinear Interpolation Backward Pass

## Overview
Trilinear Gradient Interpolation computes the gradient of the trilinear interpolation function with respect to its input data values, enabling the backpropagation of loss gradients through a differentiable volume rendering or sampling pipeline [S1, unofficial]. It solves the problem of efficiently distributing gradients from output samples back to the source voxel payloads in sparse voxel octrees (SVOs) without recomputing the forward interpolation logic [S1, unofficial]. Unlike [Normal Approximation with Finite Differences](9736-normal-approximation-with-finite-differences.md), which approximates gradients via finite differences of scalar fields, this method relies on the exact analytical derivatives of the trilinear kernel, ensuring smooth gradients and avoiding boundary artifacts [S1, unofficial]. It differs from [Gradient Volume Interpolation (General)](16060-gradient-volume-interpolation-general.md) by being specifically optimized for the trilinear kernel within an SVO structure, rather than providing a generic framework for any interpolation kernel [S1, unofficial].

## How it works
**Inputs:**
- `grad_outputs`: A tensor of shape `(num_points, channels)` containing the gradients of the loss with respect to the output samples [S1, unofficial].
- `points`: An array of 3D coordinates `(x, y, z)` for each point being sampled [S1, unofficial].
- `min_bound`, `max_bound`: The bounding box of the volume [S1, unofficial].
- `max_depth`: The maximum depth of the SVO [S1, unofficial].
- `nodes`: The SVO node structure (array of `NodeDescriptor` or `WideNodeDescriptor`) [S1, unofficial].
- `leaf_payload_indices`: Mapping from leaf IDs to payload indices [S1, unofficial].
- `num_leaves`: Number of leaves in the SVO [S1, unofficial].
- `channels`: Number of channels per payload (e.g., 3 for RGB, 1 for scalar) [S1, unofficial].

**Outputs:**
- `grad_payload`: A tensor of shape `(num_leaves, channels)` initialized to zeros, accumulating the gradients for each leaf payload [S1, unofficial].

**Data Structures:**
- `NodeDescriptor`: Contains `child_mask`, `leaf_mask`, `payload_base`, `child_base`, and `internal_child_mask` [S1, unofficial].
- `WideNodeDescriptor`: Similar to `NodeDescriptor` but uses 64-bit masks for wider traversal [S1, unofficial].
- `payload_index`: Integer index into `grad_payload` [S1, unofficial].

**Steps:**
1. Initialize `grad_payload` as a zero-filled array of size `num_leaves * channels` [S1, unofficial].
2. For each point `p` in `points` (index `point_index`):
   a. Check if `p` is within `min_bound` and `max_bound`. If not, skip to the next point [S1, unofficial].
   b. Compute the voxel coordinates `(voxel_x, voxel_y, voxel_z)` and fractional offsets `(frac_x, frac_y, frac_z)` relative to the grid defined by `max_depth` [S1, unofficial].
      - `scale = 2^max_depth`
      - `voxel_x = floor((p.x - min_bound.x) / (max_bound.x - min_bound.x) * scale)`
      - `frac_x = ((p.x - min_bound.x) / (max_bound.x - min_bound.x) * scale) - voxel_x`
      - Similarly for `y` and `z` [S1, unofficial].
   c. Traverse the SVO to find the leaf node containing `p`. If no leaf is found, skip [S1, unofficial].
   d. Retrieve the `payload_index` for the leaf [S1, unofficial]. If `payload_index < 0`, skip [S1, unofficial].
   e. Compute the trilinear interpolation weights for the 8 corners of the unit cube surrounding `p` [S1, unofficial].
      - Let `fadeX = frac_x`, `fadeY = frac_y`, `fadeZ = frac_z` [S1, unofficial].
      - The 8 corners are indexed by `(L/H, L/H, L/H)` where L=0, H=1.
      - Compute intermediate gradients along X:
        - `x1 = lerp(grad_outputs[point_index, :], grad_outputs[point_index, :], fadeX)`? No, this is backward pass.
        - Actually, the backward pass distributes `grad_outputs` to the 8 corners based on the derivative of the trilinear function [S1, unofficial].
        - The derivative of trilinear interpolation with respect to a corner value is the product of the weights for the other dimensions and the complementary weight for the current dimension.
        - Let `w_L = 1 - fade`, `w_H = fade` [S1, unofficial].
        - For corner `(cx, cy, cz)`:
          - `weight = (cx ? fadeX : (1-fadeX)) * (cy ? fadeY : (1-fadeY)) * (cz ? fadeZ : (1-fadeZ))` [S1, unofficial].
          - The gradient contribution to this corner is `weight * grad_outputs[point_index, :]` [S1, unofficial].
   f. Accumulate the gradient contributions to the corresponding leaf payloads [S1, unofficial].
      - For each of the 8 corners, determine the neighbor voxel and its `payload_index` [S1, unofficial].
      - If `payload_index >= 0`, update `grad_payload[payload_index * channels + channel] += weight * grad_outputs[point_index * channels + channel]` for each channel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_gradient_interpolation(grad_outputs, points, min_bound, max_bound, max_depth, nodes, leaf_payload_indices, num_leaves, channels):
    num_points = len(points)
    grad_payload = [0.0] * (num_leaves * channels)
    
    scale = 1 << max_depth
    extent_x = max_bound.x - min_bound.x
    extent_y = max_bound.y - min_bound.y
    extent_z = max_bound.z - min_bound.z
    
    for point_index in range(num_points):
        p = points[point_index]
        
        # Check bounds
        if not (min_bound.x <= p.x <= max_bound.x and 
                min_bound.y <= p.y <= max_bound.y and 
                min_bound.z <= p.z <= max_bound.z):
            continue
            
        # Compute voxel coordinates and fractional offsets
        voxel_x = floor(((p.x - min_bound.x) / extent_x) * scale)
        voxel_y = floor(((p.y - min_bound.y) / extent_y) * scale)
        voxel_z = floor(((p.z - min_bound.z) / extent_z) * scale)
        
        frac_x = ((p.x - min_bound.x) / extent_x) * scale - voxel_x
        frac_y = ((p.y - min_bound.y) / extent_y) * scale - voxel_y
        frac_z = ((p.z - min_bound.z) / extent_z) * scale - voxel_z
        
        # Traverse SVO to find leaf (simplified, assuming query_single_point_wide_device logic)
        # In practice, this involves traversing the node hierarchy
        # For the purpose of this article, we assume we get the 8 corner payload indices
        # If the point is inside a leaf, the 8 corners might be in adjacent leaves or the same leaf
        
        # Compute weights for each of the 8 corners
        # Corners: (0,0,0) to (1,1,1)
        corners = [
            (0, 0, 0), (1, 0, 0), (0, 1, 0), (1, 1, 0),
            (0, 0, 1), (1, 0, 1), (0, 1, 1), (1, 1, 1)
        ]
        
        for cx, cy, cz in corners:
            # Compute weight for this corner
            wx = frac_x if cx == 1 else (1.0 - frac_x)
            wy = frac_y if cy == 1 else (1.0 - frac_y)
            wz = frac_z if cz == 1 else (1.0 - frac_z)
            weight = wx * wy * wz
            
            # Determine the voxel coordinates for this corner
            corner_voxel_x = voxel_x + cx
            corner_voxel_y = voxel_y + cy
            corner_voxel_z = voxel_z + cz
            
            # Find payload index for this corner voxel
            # This requires SVO traversal for each corner
            payload_index = find_payload_index_for_voxel(corner_voxel_x, corner_voxel_y, corner_voxel_z, min_bound, max_bound, max_depth, nodes, leaf_payload_indices)
            
            if payload_index < 0:
                continue
                
            # Accumulate gradient
            base_idx = payload_index * channels
            for c in range(channels):
                grad_payload[base_idx + c] += weight * grad_outputs[point_index * channels + c]
                
    return grad_payload

def find_payload_index_for_voxel(voxel_x, voxel_y, voxel_z, min_bound, max_bound, max_depth, nodes, leaf_payload_indices):
    # Simplified traversal logic similar to query_single_point_wide_device
    # Returns -1 if not found
    # This is a placeholder for the actual SVO traversal code
    # In the actual implementation, this would traverse the node hierarchy
    # and return the payload index of the leaf containing the voxel
    pass
```

**Edge Cases and Pitfalls:**
- Points outside the volume bounds are skipped [S1, unofficial].
- If a corner voxel does not have a leaf (empty space), its gradient contribution is ignored [S1, unofficial].
- Precision issues may arise if `frac_x`, `frac_y`, or `frac_z` are very close to 0 or 1, leading to numerical instability in weight calculations [G].
- The SVO traversal must be consistent with the forward pass to ensure correct gradient flow [S1, unofficial].

## Subtopics

### [Trilinear Normal Interpolation](4892-trilinear-normal-interpolation.md)
Trilinear Normal Interpolation computes the surface normal at an arbitrary point within a voxel by trilinearly interpolating pre-computed normal vectors stored at the voxel's eight corners. It solves the problem of shading discontinuities that arise when using flat, per-voxel normals or when interpolating scalar fields separately from their derivatives. Unlike [Normal Approximation with Finite Differences](9736-normal-approximation-with-finite-differences.md), which computes gradients on-the-fly from scalar data, this method relies on a pre-computed grid of normals, offering a trade-off between memory usage and runtime gradient calculation cost.

### [Normal Approximation with Finite Differences](9736-normal-approximation-with-finite-differences.md)
Normal Approximation with Finite Differences computes the surface normal at a ray-scalar field intersection point by approximating the gradient via finite differences of the scalar field values. It differs from its parent method, Trilinear Gradient Interpolation, by explicitly requiring trilinear interpolation calls to evaluate the scalar field at displaced points, rather than relying on pre-computed or stored gradients. This approach avoids discontinuities at voxel boundaries that can cause artifacts in shading.

### [Gradient Volume Interpolation (General)](16060-gradient-volume-interpolation-general.md)
Gradient volume interpolation computes the spatial gradient of a scalar field at arbitrary positions within a 3D grid by interpolating pre-computed gradient values stored in a dedicated texture or volume buffer. It solves the problem of obtaining smooth, continuous surface normals for shading in volume rendering, avoiding the artifacts inherent in finite difference approximations. It differs from [Trilinear Gradient Interpolation](3470-trilinear-gradient-interpolation.md) by providing a general framework for gradient access, where the specific interpolation kernel (e.g., trilinear) is a parameter, and from [Normal Approximation with Finite Differences](9736-normal-approximation-with-finite-differences.md) by relying on pre-stored data rather than on-the-fly differentiation.

## Operating characteristics
- **Time Complexity:** O(N * D) where N is the number of points and D is the depth of the SVO for traversal, plus O(N * 8 * C) for gradient accumulation, where C is the number of channels [S1, unofficial].
- **Memory Complexity:** O(L * C) for storing `grad_payload`, where L is the number of leaves [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of points and leaves [S1, unofficial].
- **Strengths:** Exact gradient computation, efficient for sparse data structures [S1, unofficial].
- **Limitations:** Requires SVO traversal for each corner, which can be costly if the tree is deep [G].
- **When to use:** When differentiable volume rendering is required and the data is stored in an SVO [S1, unofficial].
- **When not to use:** When the data is dense and stored in a regular grid, or when approximate gradients are sufficient [G].

## Background
Trilinear Gradient Interpolation is a method used in differentiable volume rendering, particularly in the context of Sparse Voxel Octrees (SVOs) [S1, unofficial]. It was developed to enable gradient-based optimization of volume rendering parameters, such as material properties or geometry [S1, unofficial]. The method is based on the analytical derivatives of the trilinear interpolation function [S1, unofficial].

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
[S2] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
