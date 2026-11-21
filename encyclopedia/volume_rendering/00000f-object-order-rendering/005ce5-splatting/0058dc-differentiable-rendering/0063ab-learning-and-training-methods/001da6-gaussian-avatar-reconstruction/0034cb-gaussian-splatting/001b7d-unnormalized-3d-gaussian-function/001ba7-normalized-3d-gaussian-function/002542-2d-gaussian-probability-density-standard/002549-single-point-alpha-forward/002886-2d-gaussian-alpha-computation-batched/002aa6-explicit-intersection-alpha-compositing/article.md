# Explicit Intersection Alpha Compositing

Also called: Alpha Compositing Rendering, accumulate

## Overview
Explicit Intersection Alpha Compositing is a method for rendering 2D Gaussian splats by processing individual pixel-Gaussian intersection pairs rather than iterating over pixels or Gaussians in a nested loop. It solves the problem of arbitrary intersection patterns in differentiable rendering by explicitly enumerating all overlaps between pixels and Gaussians, allowing for efficient batched computation and automatic differentiation [S1, unofficial]. Unlike standard rasterization approaches that rely on fixed tile-based sorting or implicit accumulation, this method relies on explicit pairs `(gaussian_id, pixel_id, image_id)` to handle complex overlaps and supports the derivation of gradients for the backward pass [S1, unofficial].

## How it works
**Inputs:**
- `image_ids`: Array of shape $[M]$ containing the image index for each intersection.
- `pixel_ids_x`, `pixel_ids_y`: Arrays of shape $[M]$ containing the integer pixel coordinates for each intersection.
- `gaussian_ids`: Array of shape $[M]$ containing the index of the Gaussian for each intersection.
- `means2d`: Array of shape $[N, 2]$ containing the 2D means of $N$ Gaussians.
- `conics`: Array of shape $[N, 3]$ containing the inverse covariance matrix elements (upper triangle) for each Gaussian. Layout: `[conic00, conic01, conic22]`.
- `opacities`: Array of shape $[N]$ containing the opacity $\alpha_i$ for each Gaussian.
- `colors`: Array of shape $[N, C]$ containing the color values for each Gaussian (e.g., $C=3$ for RGB).
- `image_height`, `image_width`: Dimensions of the output image.
- `MAX_ALPHA`: Scalar clamping value for per-intersection alpha (typically $< 1.0$ to prevent instability).

**Outputs:**
- `renders`: Array of shape $[H, W, C]$ containing the final composited colors.
- `alphas_out`: Array of shape $[H, W, 1]$ containing the final composited alpha values.

**Data Structures:**
- `deltas`: Temporary array of shape $[M, 2]$ storing the vector from the Gaussian mean to the pixel center.
- `sigmas`: Temporary array of shape $[M]$ storing the Gaussian exponent values.
- `alphas_per_isect`: Temporary array of shape $[M]$ storing the unclamped or clamped alpha contribution of each intersection.
- `ray_indices`: Temporary array of shape $[M]$ mapping each intersection to a unique linear pixel index in the flattened image.
- `weights`: Temporary array of shape $[M]$ storing the final compositing weight for each intersection.
- `_transmittances`: Temporary array of shape $[M]$ storing the transmittance at each intersection (used for gradient computation).

**Steps:**

1.  **Compute Pixel Coordinates:**
    Convert integer pixel IDs to floating-point coordinates by adding 0.5 to center them in the pixel.
    $$ \text{pixel\_coords} = [\text{pixel\_ids\_x} + 0.5, \text{pixel\_ids\_y} + 0.5] \in \mathbb{R}^{M \times 2} $$

2.  **Gather Gaussian Parameters:**
    Using `gaussian_ids`, gather the corresponding `means2d`, `conics`, `opacities`, and `colors` for all $M$ intersections. Let these be `means2d_selected`, `conics_selected`, `opacities_selected`, and `colors_selected`.

3.  **Compute Distances (Deltas):**
    Calculate the vector from each Gaussian's 2D mean to the corresponding pixel coordinate.
    $$ \text{deltas} = \text{pixel\_coords} - \text{means2d\_selected} \in \mathbb{R}^{M \times 2} $$
    Let $\delta_x = \text{deltas}[:, 0]$ and $\delta_y = \text{deltas}[:, 1]$.

4.  **Compute Gaussian Exponent (Sigma):**
    Compute the exponent $\sigma_i$ for the 2D Gaussian function using the conic matrix elements and the deltas. The conic matrix $\Sigma'^{-1}$ is represented by `conics_selected` as $[c_{00}, c_{01}, c_{22}]$.
    $$ \sigma_i = 0.5 \left( c_{00} \delta_x^2 + c_{22} \delta_y^2 + 2 c_{01} \delta_x \delta_y \right) $$
    In code:
    ```python
    sigmas = 0.5 * (conics_selected[:, 0] * deltas[:, 0]**2 + 
                    conics_selected[:, 2] * deltas[:, 1]**2 + 
                    2 * conics_selected[:, 1] * deltas[:, 0] * deltas[:, 1])
    ```
    *Note: The source code [S1, unofficial] uses `conics_selected[:, 1] * deltas[:, 0] * deltas[:, 1]` without the factor of 2, implying the input conics might store $2 \times$ the off-diagonal term or the formula is adapted. We follow the source code exactly.*
    Corrected based on source code [S1, unofficial]:
    $$ \sigma_i = 0.5 \left( c_{00} \delta_x^2 + c_{22} \delta_y^2 + c_{01} \delta_x \delta_y \right) $$

5.  **Compute Per-Intersection Alpha:**
    Calculate the alpha contribution for each intersection, clamped to `MAX_ALPHA`.
    $$ \alpha_i^{\text{isect}} = \min(\alpha_i \cdot e^{-\sigma_i}, \text{MAX\_ALPHA}) $$
    where $\alpha_i$ is the Gaussian's opacity.

6.  **Compute Ray Indices:**
    Map each intersection to a unique linear index in the flattened image space to group intersections by pixel.
    $$ \text{ray\_indices} = \text{image\_ids} \times (H \times W) + \text{pixel\_ids\_y} \times W + \text{pixel\_ids\_x} $$
    *Note: The source code [S1, unofficial] uses `image_ids * (image_height * image_width) + pixel_ids`. Assuming `pixel_ids` in the source refers to a flattened pixel index or the code assumes a specific layout. Standard flattening is $y \cdot W + x$. If `pixel_ids` in the source is a single array of flattened indices, the formula is:*
    $$ \text{ray\_indices} = \text{image\_ids} \times (H \times W) + \text{pixel\_ids\_flat} $$

7.  **Compute Compositing Weights:**
    Sort the intersections for each ray (pixel) by depth (not explicitly detailed in the snippet, but implied by the need for front-to-back or back-to-front accumulation). Then, compute the weights $w_i$ using the standard alpha compositing recurrence.
    $$ w_i = \alpha_i^{\text{isect}} \prod_{j=1}^{i-1} (1 - \alpha_j^{\text{isect}}) $$
    This is typically implemented via a cumulative product of transmittances. Let $T_0 = 1$ and $T_i = T_{i-1} (1 - \alpha_i^{\text{isect}})$. Then $w_i = T_{i-1} \alpha_i^{\text{isect}}$.
    The function `render_weight_from_alpha` performs this step, returning `weights` and `_transmittances`.

8.  **Accumulate Colors:**
    Accumulate the weighted colors for each pixel.
    $$ C(x) = \sum_{i \in \text{pixels}(x)} w_i \cdot \text{colors\_selected}_i $$
    This is done using `accumulate_along_rays(weights, colors_selected, ray_indices, total_pixels)`.

9.  **Accumulate Alpha:**
    Accumulate the weights to get the final alpha channel (since $\sum w_i = 1 - T_N$).
    $$ A(x) = \sum_{i \in \text{pixels}(x)} w_i $$
    This is done using `accumulate_along_rays(weights, None, ray_indices, total_pixels)`.

10. **Reshape Outputs:**
    Reshape the accumulated arrays to the final image dimensions $[H, W, C]$ and $[H, W, 1]$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def explicit_intersection_alpha_compositing(
    image_ids, pixel_ids_x, pixel_ids_y, gaussian_ids,
    means2d, conics, opacities, colors,
    image_height, image_width, MAX_ALPHA
):
    M = len(image_ids)
    
    # 1. Compute Pixel Coordinates
    pixel_coords = mx.stack([pixel_ids_x, pixel_ids_y], axis=-1).astype(mx.float32) + 0.5
    
    # 2. Gather Gaussian Parameters
    means2d_selected = means2d[gaussian_ids]
    conics_selected = conics[gaussian_ids]
    opacities_selected = opacities[gaussian_ids]
    colors_selected = colors[gaussian_ids]
    
    # 3. Compute Deltas
    deltas = pixel_coords - means2d_selected
    
    # 4. Compute Sigmas
    # conics_selected: [c00, c01, c22]
    sigmas = 0.5 * (
        conics_selected[:, 0] * deltas[:, 0]**2 + 
        conics_selected[:, 2] * deltas[:, 1]**2 + 
        conics_selected[:, 1] * deltas[:, 0] * deltas[:, 1]
    )
    
    # 5. Compute Per-Intersection Alpha
    alphas_per_isect = mx.minimum(opacities_selected * mx.exp(-sigmas), MAX_ALPHA)
    
    # 6. Compute Ray Indices
    # Assuming pixel_ids_x and pixel_ids_y are needed for flat index
    pixel_ids_flat = pixel_ids_y * image_width + pixel_ids_x
    ray_indices = image_ids * (image_height * image_width) + pixel_ids_flat
    
    # 7. Compute Weights and Transmittances
    # This step requires sorting intersections per ray by depth (z-order)
    # and then applying front-to-back compositing.
    # The source [S1, unofficial] abstracts this into render_weight_from_alpha.
    # We assume a helper function exists that handles the sorting and recurrence.
    weights, _transmittances = render_weight_from_alpha(
        alphas_per_isect, ray_indices, image_height * image_width
    )
    
    # 8. Accumulate Colors
    renders_flat = accumulate_along_rays(weights, colors_selected, ray_indices, image_height * image_width)
    
    # 9. Accumulate Alpha
    alphas_flat = accumulate_along_rays(weights, None, ray_indices, image_height * image_width)
    
    # 10. Reshape
    renders = renders_flat.reshape(image_height, image_width, colors.shape[1])
    alphas_out = alphas_flat.reshape(image_height, image_width, 1)
    
    return renders, alphas_out
```

**Edge Cases and Pitfalls:**
- **Sorting:** The correctness of alpha compositing depends on the correct ordering of Gaussians along the view ray (front-to-back or back-to-front). The source [S1, unofficial] does not detail the sorting mechanism, but it is a critical prerequisite for `render_weight_from_alpha`.
- **Precision:** Clamping alpha to `MAX_ALPHA` prevents numerical instability when multiple Gaussians overlap significantly, but choosing a value too close to 1.0 can lead to incorrect blending if the sum of alphas exceeds 1.0 locally.
- **Empty Pixels:** Pixels with no intersections will have zero values in the accumulated arrays, which is correct for transparent backgrounds.

## Subtopics

### [Gamma-Space Alpha Blending](7704-gamma-space-alpha-blending.md)
Gamma-Space Alpha Blending performs Gaussian alpha compositing directly in the Gamma color space to match the training conditions of standard 3D Gaussian Splatting (3DGS) assets. It solves the quality degradation caused by converting colors to linear space before blending, which is the default behavior in modern linear-rendering pipelines. Unlike its parent method, Explicit Intersection Alpha Compositing, which handles arbitrary intersection patterns via explicit pairs, this method assumes a standard front-to-back or back-to-front accumulation where the entire splatting pass occurs within a single Gamma-space context.

### [Capacity Response Modeling](14401-capacity-response-modeling.md)
Capacity Response Modeling is a pruning method that computes individual importance scores and pairwise interaction terms for Gaussian selection by measuring RGB changes when Gaussian opacity varies. It determines which $K_b$ members to retain in each block by accounting for joint rendering effects via the response matrix $R_b$. It builds on [Explicit Intersection Alpha Compositing](4814-explicit-intersection-alpha-compositing.md) by using its alpha compositing logic to derive gradients for selection, differing from [Gamma-Space Alpha Blending](7704-gamma-space-alpha-blending.md) which operates in a different color space.

## Operating characteristics
- **Time Complexity:** $O(M)$, where $M$ is the number of intersections. The dominant cost is the gathering of parameters and the computation of sigmas and alphas, which are element-wise operations. The sorting and accumulation steps are also linear in $M$ if efficient bucketing or sorting is used.
- **Memory Complexity:** $O(M)$ for storing the intersection arrays. The output size is $O(H \times W)$.
- **Performance:** This method is highly parallelizable on GPUs due to the explicit intersection list. It avoids the overhead of nested loops and is suitable for differentiable rendering pipelines.
- **Strengths:** Handles arbitrary intersection patterns efficiently; supports automatic differentiation; flexible for various Gaussian distributions.
- **Limitations:** Requires explicit generation of the intersection list, which can be memory-intensive for scenes with many overlapping Gaussians. Sorting per pixel is required for correct compositing.

## Background
The method is derived from the 2D Gaussian Splatting (2DGS) framework, which extends 3D Gaussian Splatting by using 2D Gaussians for more flexible shape representation. The explicit intersection approach is a key component of differentiable rasterizers for Gaussian splatting, allowing for efficient gradient computation. The specific implementation details are found in the `gsplat-mlx` library [S1, unofficial] and the AtomGS paper [S2, unofficial].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
