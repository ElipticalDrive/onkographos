# Explicit Intersection Alpha Compositing

Also called: accumulate

## Overview
Explicit Intersection Alpha Compositing is the final rasterization pass in the 2D Gaussian Splatting pipeline, responsible for combining the pre-computed 2D Gaussian alpha values into a final RGB image and an alpha mask [S1, unofficial]. It solves the problem of correctly blending overlapping splats along each pixel's viewing ray using front-to-back compositing [S1, unofficial]. It is distinguished from parent methods by operating on explicit intersection pairs `(gaussian_id, pixel_id, image_id)` rather than iterating over pixels or Gaussians, allowing arbitrary intersection patterns and leveraging automatic differentiation for the backward pass [S1, unofficial].

## How it works
**Inputs:** `pixel_ids_x` and `pixel_ids_y` (integer arrays of shape $[M]$), `image_ids` (integer array of shape $[M]$), `gaussian_ids` (integer array of shape $[M]$), `means2d` (array of shape $[N, 2]$), `conics` (array of shape $[N, 3]$), `opacities` (array of shape $[N]$), `colors` (array of shape $[N, C]$), `image_height` and `image_width` (integers), and `MAX_ALPHA` (scalar float) [S1, unofficial].

**Outputs:** `renders` (array of shape $[H, W, C]$) and `alphas_out` (array of shape $[H, W, 1]$) [S1, unofficial].

**Data Structures:** The method relies on a flat list of intersections of size $M$ [S1, unofficial]. Each entry $i$ represents one Gaussian splatting onto one pixel [S1, unofficial]. The arrays `pixel_ids`, `image_ids`, and `gaussian_ids` are parallel arrays indexing into the global Gaussian attributes [S1, unofficial].

**Steps:**

1.  **Compute Pixel Coordinates:** Convert flat or structured pixel IDs into 2D coordinates [S1, unofficial].
    $$ \text{pixel\_coords} = [\text{pixel\_ids}_x, \text{pixel\_ids}_y] + 0.5 $$
    Shape: $[M, 2]$ [S1, unofficial]. The $+0.5$ centers the coordinate within the pixel [S1, unofficial].

2.  **Gather Gaussian Parameters:** Using `image_ids` and `gaussian_ids`, gather the corresponding `means2d`, `conics`, `opacities`, and `colors` for each intersection [S1, unofficial].

3.  **Compute Gaussian Exponent (Sigma):** Calculate the exponent of the 2D Gaussian function at the pixel center [S1, unofficial]. Let $\Delta = \text{pixel\_coords} - \text{means}_{sel}$ be the displacement vector $[\Delta_x, \Delta_y]$ [S1, unofficial].
    $$ \sigma_i = 0.5 \cdot (\text{conics}_{i,0} \Delta_{i,x}^2 + \text{conics}_{i,2} \Delta_{i,y}^2) + \text{conics}_{i,1} \Delta_{i,x} \Delta_{i,y} $$
    Shape: $[M]$ [S1, unofficial].

4.  **Compute Per-Intersection Alpha:** Calculate the alpha value for each intersection, clamped to `MAX_ALPHA` [S1, unofficial].
    $$ \alpha_i = \min(\text{opacities}_{sel, i} \cdot e^{-\sigma_i}, \text{MAX\_ALPHA}) $$
    Shape: $[M]$ [S1, unofficial].

5.  **Compute Ray Indices:** Map each intersection to a unique ray index for compositing [S1, unofficial]. For a single image batch, this is simply the flattened pixel index [S1, unofficial].
    $$ \text{ray\_index}_i = \text{image\_ids}_i \cdot (H \cdot W) + \text{pixel\_ids}_i $$
    Shape: $[M]$ [S1, unofficial].

6.  **Compute Compositing Weights:** Perform front-to-back alpha compositing to determine the weight of each splat [S1, unofficial]. This involves computing transmittance $T$ and weight $w$ [S1, unofficial]. For a ray with intersections sorted front-to-back:
    $$ T_0 = 1 $$
    $$ w_i = \alpha_i \prod_{j < i} (1 - \alpha_j) $$
    $$ T_{i+1} = T_i (1 - \alpha_i) $$
    The function `render_weight_from_alpha` returns `weights` ($w_i$) and `_transmittances` (unused in final output but used for gradients) [S1, unofficial].

7.  **Accumulate Colors:** Sum the weighted colors for each ray [S1, unofficial].
    $$ \text{render}_r = \sum_{i \in \text{ray } r} w_i \cdot \text{colors}_{sel, i} $$

8.  **Accumulate Alpha:** Sum the weights for each ray to get the final alpha (since $\sum w_i = 1 - T_{final}$) [S1, unofficial].
    $$ \alpha_{out, r} = \sum_{i \in \text{ray } r} w_i $$

9.  **Reshape Outputs:** Reshape the accumulated arrays from flat ray indices to image dimensions [S1, unofficial].
    $$ \text{renders} \in \mathbb{R}^{H \times W \times C} $$
    $$ \text{alphas\_out} \in \mathbb{R}^{H \times W \times 1} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def explicit_intersection_alpha_compositing(
    pixel_ids_x, pixel_ids_y, image_ids, gaussian_ids,
    means2d, conics, opacities, colors,
    image_height, image_width, MAX_ALPHA=1.0
):
    M = len(pixel_ids_x)
    total_pixels = image_height * image_width
    C = len(colors[0])

    # 1. Compute pixel coordinates
    pixel_coords = [(float(x) + 0.5, float(y) + 0.5) for x, y in zip(pixel_ids_x, pixel_ids_y)]

    # 2. Gather parameters
    means2d_sel = [means2d[g] for g in gaussian_ids]
    conics_sel = [conics[g] for g in gaussian_ids]
    opacities_sel = [opacities[g] for g in gaussian_ids]
    colors_sel = [colors[g] for g in gaussian_ids]

    # 3. Compute sigma
    sigmas = []
    for i in range(M):
        dx = pixel_coords[i][0] - means2d_sel[i][0]
        dy = pixel_coords[i][1] - means2d_sel[i][1]
        c0, c1, c2 = conics_sel[i]
        sigmas.append(0.5 * (c0 * dx**2 + c2 * dy**2) + c1 * dx * dy)

    # 4. Compute alpha
    alphas_per_isect = [min(o * math.exp(-s), MAX_ALPHA) for o, s in zip(opacities_sel, sigmas)]

    # 5. Compute ray indices
    ray_indices = [int(img) * total_pixels + int(px) for img, px in zip(image_ids, pixel_ids_x)]

    # 6. Compute weights (front-to-back compositing)
    # Assumes intersections are sorted front-to-back per ray [G]
    ray_groups = {}
    for i in range(M):
        ray_groups.setdefault(ray_indices[i], []).append(i)

    weights = [0.0] * M
    for r, indices in ray_groups.items():
        T = 1.0
        for i in indices:
            alpha = alphas_per_isect[i]
            weights[i] = alpha * T
            T *= (1.0 - alpha)

    # 7 & 8. Accumulate colors and alpha
    renders_flat = [0.0] * (total_pixels * C)
    alphas_flat = [0.0] * total_pixels

    for i in range(M):
        r = ray_indices[i]
        w = weights[i]
        alphas_flat[r] += w
        for c in range(C):
            renders_flat[r * C + c] += w * colors_sel[i][c]

    # 9. Reshape outputs
    renders = [[renders_flat[r * C + c] for c in range(C)] for r in range(total_pixels)]
    renders = [renders[r * image_width:(r + 1) * image_width] for r in range(image_height)]
    alphas_out = [[alphas_flat[r * image_width + c]] for r in range(total_pixels)]
    alphas_out = [alphas_out[r * image_width:(r + 1) * image_width] for r in range(image_height)]

    return renders, alphas_out
```

**Parameters:**
- `MAX_ALPHA`: Clamping value for alpha to prevent $1 - \alpha$ from becoming zero, which causes division by zero or NaN in gradients [S1, unofficial]. Typical value: $0.99$ or $1.0$ [G].

**Edge Cases and Pitfalls:**
- **Precision:** The exponential $e^{-\sigma}$ can underflow for large $\sigma$ [G]. This is handled by the clamping of alpha [S1, unofficial].
- **Sorting:** The correctness of front-to-back compositing depends on the intersections being sorted by depth (Z-coordinate) within each ray [G]. If the rasterization step does not provide sorted intersections, the compositing will be incorrect [G].
- **Boundaries:** Pixel coordinates must be within $[0, H-1] \times [0, W-1]$ [G]. Intersections outside the image bounds should be filtered out before this step [G].

## Operating characteristics
- **Time Complexity:** $O(M)$ where $M$ is the number of intersections [S1, unofficial]. The dominant cost is the gather operations and the compositing reduction, which is linear in the number of splats per pixel [S1, unofficial].
- **Memory Complexity:** $O(M)$ for storing the intersection arrays and intermediate weights [S1, unofficial].
- **Performance:** Highly parallelizable on GPUs [G]. The explicit intersection list allows for efficient memory access patterns compared to pixel-order rendering when the scene is sparse [G].
- **Strengths:** Supports arbitrary intersection patterns [S1, unofficial]. Enables automatic differentiation for end-to-end training [S1, unofficial]. Efficient for sparse scenes [G].
- **Limitations:** Requires sorting intersections per ray for correct compositing [G]. Memory usage scales with the number of splats per pixel [G].

## Background
This method is part of the 2D Gaussian Splatting pipeline, specifically the rendering stage [S1, unofficial]. It builds on the 2D Gaussian Alpha Computation (Batched) method [S1, unofficial]. The approach of explicit intersection compositing is common in differentiable rendering pipelines to allow gradients to flow through the compositing operation [G].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
