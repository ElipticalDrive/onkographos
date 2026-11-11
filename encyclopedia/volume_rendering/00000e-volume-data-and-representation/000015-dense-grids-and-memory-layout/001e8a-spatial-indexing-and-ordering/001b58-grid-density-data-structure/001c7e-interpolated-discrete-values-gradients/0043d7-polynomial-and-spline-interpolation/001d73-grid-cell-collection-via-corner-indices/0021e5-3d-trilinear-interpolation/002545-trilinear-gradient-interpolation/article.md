# Trilinear Gradient Interpolation

Also called: Trilinear Interpolation Backward Pass

## Overview
Trilinear Gradient Interpolation computes the gradient or surface normal at an arbitrary point within a voxel by interpolating pre-computed vector values stored at the eight corners of the surrounding unit cube [S3, unofficial]. It solves the problem of obtaining smooth, continuous directional derivatives from discrete grid data, which is essential for shading and ray marching in volume rendering [S3, unofficial]. It differs from standard [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md) by operating on vector fields rather than scalar values, and from [Cubic Volume Interpolation](4278-cubic-volume-interpolation.md) by using only corner data without higher-order derivative constraints [S3, unofficial]. Trilinear Gradient Interpolation extends standard trilinear interpolation to vector fields [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A 3D grid of pre-computed gradient or normal vectors at voxel corners; the fractional coordinates $(t_x, t_y, t_z)$ of the query point relative to the voxel's minimum corner, where $t \in [0, 1]$ [S3, unofficial]. For training workflows, the upstream gradients of the loss with respect to the interpolated output, `grad_outputs`, of size `points.size() * channels` [S1, unofficial].
- **Outputs:** The interpolated gradient/normal vector $G_{interp}$ (forward pass); updated `grad_payload` array with accumulated gradients (backward pass) [S1, unofficial].

**Data Structures:**
- The gradient grid is stored as three separate 3D arrays (one for each component $x, y, z$) or a single 4D array with shape $(W, H, D, 3)$ [S3, unofficial].
- Indices are integer coordinates $(i, j, k)$ identifying the voxel corner, where $0 \le i < W, 0 \le j < H, 0 \le k < D$ [S3, unofficial].
- The backward pass accumulates gradients into a `grad_payload` vector of size `payload_rows * channels`, initialized to zeros [S1, unofficial].

**Numbered Steps:**
1. **Retrieve Corner Gradients:** Identify the 8 corner gradients surrounding the query point [S2, unofficial]. Let $G_{ijk}$ be the gradient vector at corner $(i,j,k)$, where $i,j,k \in \{0,1\}$ represent the relative position within the voxel (0 for min, 1 for max) [S3, unofficial].
2. **Compute Interpolation Weights:** Calculate the trilinear weights for each corner based on the fractional coordinates [S3, unofficial].
3. **Interpolate Components:** Compute the interpolated gradient vector $G_{interp}$ by summing the weighted corner gradients, equivalently implemented as three sequential linear interpolations (lerps) along the x, y, and z axes [S2, unofficial].
4. **Normalization (Optional):** If the input gradients are unit normals, normalize the interpolated result to maintain unit length [S3, unofficial].
5. **Backward Pass (Gradient Distribution):** Initialize `grad_payload` to zeros [S1, unofficial]. Loop over each point. If outside root bounds, skip. Compute weights and total weight. If weight is 0, continue. Determine neighbor voxel and lookup its `payload_index` [S1, unofficial]. If `payload_index < 0`, continue. Get pointer to gradient row: `grad_row = grad_payload.data() + payload_index * channels` [S1, unofficial]. Loop over each channel. Update gradient: `grad_row[channel] += weight * grad_outputs[point_index * channels + channel]` [S1, unofficial].

**Formulas:**
The weight for corner $(i,j,k)$ is:
$$w_{ijk} = (1-t_x)^{1-i} t_x^i (1-t_y)^{1-j} t_y^j (1-t_z)^{1-k} t_z^k$$
where $t_x, t_y, t_z \in [0, 1]$ are the fractional coordinates and $i,j,k \in \{0,1\}$ are the corner indices [S3, unofficial].
The interpolated vector is computed as:
$$G_{interp} = \sum_{i=0}^{1} \sum_{j=0}^{1} \sum_{k=0}^{1} w_{ijk} G_{ijk}$$
which expands to the sequential lerp operations:
- X-axis: $x_1 = \text{lerp}(G_{000}, G_{100}, t_x)$, $x_2 = \text{lerp}(G_{010}, G_{110}, t_x)$, $x_3 = \text{lerp}(G_{001}, G_{101}, t_x)$, $x_4 = \text{lerp}(G_{011}, G_{111}, t_x)$ [S2, unofficial]
- Y-axis: $y_1 = \text{lerp}(x_1, x_2, t_y)$, $y_2 = \text{lerp}(x_3, x_4, t_y)$ [S2, unofficial]
- Z-axis: $G_{interp} = \text{lerp}(y_1, y_2, t_z)$ [S2, unofficial]
For normalization: $N = G_{interp} / \|G_{interp}\|$ [S3, unofficial].

**Parameters:**
- $t_x, t_y, t_z$: Fractional coordinates within the voxel. Must be in $[0, 1]$. If outside, clamp or handle boundary conditions [G].
- `normalize`: Boolean flag. If true, the output is a unit vector. Default: true for normals [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Vector:** If the interpolated gradient is zero, normalization fails. Return a default vector (e.g., $[0,0,0]$ or up vector) [G].
- **Boundary Conditions:** If the query point is outside the grid, indices must be clamped or the lookup returns -1, skipping the point [S1, unofficial].
- **Precision:** Accumulating gradients in the backward pass requires careful handling of floating-point precision when many points map to the same payload index [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_gradient_forward(grid_gradients, t_x, t_y, t_z, normalize=True):
    """
    grid_gradients: list of 8 vectors [G000, G100, G010, G110, G001, G101, G011, G111]
    t_x, t_y, t_z: float in [0, 1]
    """
    G000, G100, G010, G110, G001, G101, G011, G111 = grid_gradients
    x1 = lerp(G000, G100, t_x)
    x2 = lerp(G010, G110, t_x)
    x3 = lerp(G001, G101, t_x)
    x4 = lerp(G011, G111, t_x)
    y1 = lerp(x1, x2, t_y)
    y2 = lerp(x3, x4, t_y)
    G_interp = lerp(y1, y2, t_z)
    if normalize:
        norm = vector_norm(G_interp)
        if norm > 0:
            G_interp = G_interp / norm
        else:
            G_interp = [0.0, 0.0, 0.0]
    return G_interp

def trilinear_gradient_backward(grad_outputs, weights, payload_indices, channels, grad_payload):
    """
    grad_outputs: upstream gradients of shape (num_points, channels)
    weights: scalar trilinear weights for each point
    payload_indices: integer indices into grad_payload for each point
    channels: number of channels per gradient
    grad_payload: flat array of size (payload_rows, channels)
    """
    # Initialize grad_payload to zeros
    for i in range(len(grad_payload)):
        grad_payload[i] = 0.0
    for point_idx in range(len(weights)):
        w = weights[point_idx]
        if w == 0:
            continue
        p_idx = payload_indices[point_idx]
        if p_idx < 0:
            continue
        # Get pointer to gradient row
        start = p_idx * channels
        end = start + channels
        grad_row = grad_payload[start:end]
        out_row = grad_outputs[point_idx]
        for c in range(channels):
            grad_row[c] += w * out_row[c]

def lerp(v1, v2, t):
    return [v1[i] + t * (v2[i] - v1[i]) for i in range(len(v1))]

def vector_norm(v):
    return (v[0]**2 + v[1]**2 + v[2]**2)**0.5
```

## Subtopics

### Trilinear Normal Interpolation
A specific application of Trilinear Gradient Interpolation where the grid stores surface normals instead of general gradients [S3, unofficial]. It is used in ray marching for SDFs or implicit surfaces to provide smooth shading [S3, unofficial]. It differs from general gradient interpolation by typically requiring explicit normalization of the result to maintain unit length, as interpolation of unit vectors does not preserve length [S3, unofficial]. Choose this when rendering smooth surfaces defined by implicit functions or when shading requires consistent normal directions [S3, unofficial]. Link: [Trilinear Gradient Interpolation](2614-trilinear-gradient-interpolation.md).

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, involving 8 vector retrievals and 12 vector additions/multiplications (lerps) [S3, unofficial].
- **Memory Complexity:** $O(N)$ for storing the gradient grid, where $N$ is the number of voxel corners. Requires storing 3 components per corner [S3, unofficial].
- **Performance:** Very fast, suitable for real-time rendering. The main cost is memory bandwidth for fetching 8 vectors [S3, unofficial].
- **Strengths:** Smooth gradients, easy to implement, works well with existing trilinear infrastructure [S3, unofficial].
- **Limitations:** Discontinuities at voxel boundaries if gradients are not consistent. Normalization can introduce artifacts if the interpolated vector is near zero [S3, unofficial]. Less accurate than higher-order methods like cubic interpolation for complex fields [S3, unofficial].

## Background
Trilinear interpolation is a standard technique in computer graphics for dense grids [S3, unofficial]. Trilinear Gradient Interpolation extends this to vector fields [S1, unofficial]. The method is commonly used in volume rendering for cloud shading [S1, unofficial] and in implicit surface rendering for normal estimation [S3, unofficial]. The specific implementation for normals is detailed in the SDFDiff project [S3, unofficial].

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
[S2] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S3] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
