# Explicit Intersection Alpha Compositing

Also called: Alpha Compositing Rendering, accumulate

## Overview
Explicit Intersection Alpha Compositing is the final rendering step in the 2D Gaussian Splatting pipeline, responsible for compositing pre-computed alpha values of 2D Gaussians onto the image plane to produce the final color, alpha, and normal maps [S1, unofficial]. It solves the problem of efficiently aggregating contributions from multiple Gaussians intersecting the same pixel across multiple cameras by processing explicit intersection pairs $(gaussian\_id, pixel\_id, image\_id)$, allowing for arbitrary intersection patterns [S1, unofficial]. Unlike its sibling [Gamma-Space Alpha Blending](7704-gamma-space-alpha-blending.md), which operates in perceptual color spaces, this method performs standard linear alpha compositing in the RGB space [S1, unofficial]. It builds on [2D Gaussian Alpha Computation (Batched)](4273-2d-gaussian-alpha-computation-batched.md) by utilizing the `nerfacc` library to handle the ray accumulation logic, allowing for arbitrary intersection patterns while relying on automatic differentiation for the backward pass [S1, unofficial].

## How it works
**Inputs:**
*   `image_ids`: Array of shape $[M]$ containing the image index for each intersection.
*   `pixel_ids_x`, `pixel_ids_y`: Arrays of shape $[M]$ containing the pixel coordinates for each intersection.
*   `gaussian_ids`: Array of shape $[M]$ containing the Gaussian index for each intersection.
*   `means2d_selected`: Array of shape $[M, 2]$ containing the 2D mean positions of the Gaussians.
*   `conics_selected`: Array of shape $[M, 3]$ containing the conic parameters (inverse covariance matrix elements) for the Gaussians.
*   `opacities_selected`: Array of shape $[M]$ containing the opacity values.
*   `colors_selected`: Array of shape $[M, C]$ containing the RGB colors (where $C$ is typically 3).
*   `image_height`, `image_width`: Integers defining the output image dimensions.
*   `MAX_ALPHA`: Float scalar, the maximum clamping value for per-intersection alpha.

**Outputs:**
*   `renders`: Array of shape $[H, W, C]$ containing the final composited RGB colors.
*   `alphas_out`: Array of shape $[H, W, 1]$ containing the final composited alpha values.

**Data Structures:**
*   `pixel_coords`: Array of shape $[M, 2]$, storing pixel coordinates as floats.
*   `deltas`: Array of shape $[M, 2]$, storing the distance vector from the Gaussian mean to the pixel center.
*   `sigmas`: Array of shape $[M]$, storing the Gaussian exponent values.
*   `alphas_per_isect`: Array of shape $[M]$, storing the clamped alpha for each intersection.
*   `ray_indices`: Array of shape $[M]$, mapping each intersection to a unique pixel index in the flattened image.
*   `weights`: Array of shape $[M]$, storing the front-to-back compositing weights.
*   `_transmittances`: Array of shape $[M]$, storing the transmittance values (unused in final output but computed by the accumulation function).

**Steps:**

1.  **Compute Pixel Coordinates:** Convert integer pixel IDs to floating-point coordinates. The coordinate system origin is at the top-left corner, with pixel centers at $+0.5$ offsets.
    $$ \text{pixel\_coords} = [\text{pixel\_ids\_x}, \text{pixel\_ids\_y}] + 0.5 $$
    Shape: $[M, 2]$.

2.  **Compute Distance Vectors:** Calculate the delta vector from each Gaussian's 2D mean to the corresponding pixel center.
    $$ \text{deltas} = \text{pixel\_coords} - \text{means2d\_selected} $$
    Shape: $[M, 2]$.

3.  **Compute Gaussian Exponents (Sigmas):** Calculate the exponent for the 2D Gaussian function using the conic parameters and the delta vectors. The conic matrix $C$ is represented by three values $[c_{00}, c_{01}, c_{11}]$ corresponding to the upper triangle of the symmetric $2 \times 2$ matrix.
    $$ \sigma = 0.5 \left( c_{00} \cdot \delta_x^2 + c_{11} \cdot \delta_y^2 + 2 c_{01} \cdot \delta_x \cdot \delta_y \right) $$
    Note: The source code uses `conics_selected[:, 1] * deltas[:, 0] * deltas[:, 1]` without the factor of 2, implying the input conics might already account for the factor or the formula in the source code is specific to the implementation's conic definition. Based on [S1, unofficial], the code is:
    $$ \text{sigmas} = 0.5 \cdot (c_{00} \cdot \delta_x^2 + c_{11} \cdot \delta_y^2) + c_{01} \cdot \delta_x \cdot \delta_y $$
    Shape: $[M]$.

4.  **Compute Per-Intersection Alpha:** Calculate the alpha value for each intersection by combining the Gaussian's opacity with the exponential decay, clamped to `MAX_ALPHA`.
    $$ \alpha_i = \min(\text{opacity}_i \cdot e^{-\sigma_i}, \text{MAX\_ALPHA}) $$
    Shape: $[M]$.

5.  **Compute Ray Indices:** Map each intersection to a unique linear index in the flattened image space to group intersections by pixel.
    $$ \text{ray\_index} = \text{image\_id} \cdot (H \cdot W) + \text{pixel\_id} $$
    Shape: $[M]$.

6.  **Compute Compositing Weights:** Perform front-to-back alpha compositing to determine the weight of each Gaussian contribution. This step sorts Gaussians along the ray (implicitly handled by the accumulation library or pre-sorted input) and computes the transmittance.
    $$ w_i = \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    The function `render_weight_from_alpha` returns `weights` and `_transmittances`.
    Shape: $[M]$.

7.  **Accumulate Colors:** Compute the final color for each pixel by summing the weighted colors.
    $$ C(x) = \sum_{i=1}^{N} c_i w_i $$
    Shape: $[H, W, C]$.

8.  **Accumulate Alpha:** Compute the final alpha for each pixel by summing the weights (since alpha represents the accumulated opacity).
    $$ A(x) = \sum_{i=1}^{N} w_i $$
    Shape: $[H, W, 1]$.

9.  **Reshape Outputs:** Reshape the accumulated arrays to the final image dimensions.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def explicit_intersection_alpha_compositing(
    image_ids,
    pixel_ids_x,
    pixel_ids_y,
    gaussian_ids,
    means2d_selected,
    conics_selected,
    opacities_selected,
    colors_selected,
    image_height,
    image_width,
    MAX_ALPHA=0.99
):
    """
    Perform explicit intersection alpha compositing for 2D Gaussian Splatting.
    
    Args:
        image_ids: [M] int, image indices.
        pixel_ids_x: [M] int, x-coordinates of pixels.
        pixel_ids_y: [M] int, y-coordinates of pixels.
        gaussian_ids: [M] int, Gaussian indices.
        means2d_selected: [M, 2] float, 2D means of Gaussians.
        conics_selected: [M, 3] float, conic parameters [c00, c01, c11].
        opacities_selected: [M] float, opacities.
        colors_selected: [M, C] float, colors.
        image_height: int, height of the image.
        image_width: int, width of the image.
        MAX_ALPHA: float, maximum alpha clamp.
        
    Returns:
        renders: [H, W, C] float, final RGB colors.
        alphas_out: [H, W, 1] float, final alpha map.
    """
    # Step 1: Compute pixel coordinates
    pixel_coords = mx.stack([pixel_ids_x, pixel_ids_y], axis=-1).astype(mx.float32) + 0.5  # [M, 2]
    
    # Step 2: Compute deltas
    deltas = pixel_coords - means2d_selected  # [M, 2]
    
    # Step 3: Compute sigmas
    # conics_selected: [c00, c01, c11]
    c00 = conics_selected[:, 0]
    c01 = conics_selected[:, 1]
    c11 = conics_selected[:, 2]
    dx = deltas[:, 0]
    dy = deltas[:, 1]
    
    sigmas = 0.5 * (c00 * dx**2 + c11 * dy**2) + c01 * dx * dy  # [M]
    
    # Step 4: Compute alphas per intersection
    alphas_per_isect = mx.minimum(opacities_selected * mx.exp(-sigmas), MAX_ALPHA)  # [M]
    
    # Step 5: Compute ray indices
    ray_indices = image_ids * (image_height * image_width) + pixel_ids_x  # [M]
    
    # Step 6: Compute weights and transmittances
    # Assuming render_weight_from_alpha handles sorting and front-to-back compositing
    # weights: [M], _transmittances: [M]
    weights, _transmittances = render_weight_from_alpha(alphas_per_isect, ray_indices, image_height * image_width)
    
    # Step 7: Accumulate colors
    # accumulate_along_rays sums weights * values along ray_indices
    renders_flat = accumulate_along_rays(weights, colors_selected, ray_indices, image_height * image_width)  # [H*W, C]
    
    # Step 8: Accumulate alpha
    # accumulate_along_rays with None for values sums the weights
    alphas_flat = accumulate_along_rays(weights, None, ray_indices, image_height * image_width)  # [H*W]
    
    # Step 9: Reshape outputs
    renders = renders_flat.reshape(image_height, image_width, colors_selected.shape[1])
    alphas_out = alphas_flat.reshape(image_height, image_width, 1)
    
    return renders, alphas_out
```

**Edge Cases and Pitfalls:**
*   **Precision:** The exponential decay $e^{-\sigma}$ can underflow to zero for large $\sigma$, effectively ignoring distant Gaussians. This is expected behavior but should be noted for debugging.
*   **Sorting:** The correctness of front-to-back compositing relies on the Gaussians being sorted by depth (or distance along the viewing ray) before weight computation. The `render_weight_from_alpha` function is assumed to handle this or requires pre-sorted input. If not sorted, the result will be incorrect.
*   **Conic Definition:** The formula for $\sigma$ depends on the specific definition of the conic parameters. The source code uses a specific arrangement that may differ from the standard inverse covariance matrix representation by a factor of 2 in the off-diagonal term. Implementers must verify the conic format matches the source code's expectation.
*   **MAX_ALPHA:** Clamping alpha prevents numerical instability in the product term $\prod (1-\alpha_j)$ when many overlapping Gaussians have high opacity. The value `0.99` is typical.

## Subtopics

### [Gamma-Space Alpha Blending](7704-gamma-space-alpha-blending.md)
Gamma-Space Alpha Blending performs Gaussian alpha compositing directly in the Gamma color space to match assets trained in Gamma space, avoiding quality loss from post-blend color conversion. It requires the rendering project to be configured in Gamma color space, as converting colors to linear space before blending produces incorrect results and lowers rendering quality.

### [Capacity Response Modeling](14401-capacity-response-modeling.md)
Capacity Response Modeling is a pruning method that computes individual importance scores and pairwise interaction terms for quadratic selection by measuring RGB changes when Gaussian opacity varies. It determines which $K_b$ members to retain in each block by accounting for joint rendering effects via the response matrix $R_b$.

### [2D Gaussian Accumulation (Batched)](20506-2d-gaussian-accumulation-batched.md)
2D Gaussian Accumulation (Batched) is the final rendering step in the 2D Gaussian Splatting pipeline, responsible for compositing the pre-computed alpha values of 2D Gaussians onto the image plane to produce the final color, alpha, and normal maps. It solves the problem of efficiently aggregating contributions from multiple Gaussians intersecting the same pixel across multiple cameras. Unlike its sibling [Gamma-Space Alpha Blending](7704-gamma-space-alpha-blending.md), which operates in perceptual color spaces, this method performs standard linear alpha compositing in the RGB space. It builds on [Explicit Intersection Alpha Compositing](4814-explicit-intersection-alpha-compositing.md) by utilizing the `nerfacc` library to handle the ray accumulation logic, allowing for arbitrary intersection patterns while relying on automatic differentiation for the backward pass.

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the sorting of Gaussians per pixel and the accumulation step. If $M$ is the total number of intersections and $N$ is the number of pixels, the sorting step is $O(M \log M)$ or $O(M)$ if pre-sorted. The accumulation step is $O(M)$.
*   **Memory Complexity:** The method requires storing intermediate arrays of size $M$ for `sigmas`, `alphas_per_isect`, `weights`, etc. The output size is $O(H \cdot W \cdot C)$.
*   **Performance:** Performance is highly dependent on the efficiency of the `render_weight_from_alpha` and `accumulate_along_rays` functions, which are typically implemented in optimized C++ or CUDA kernels via libraries like `nerfacc`.
*   **Strengths:** Supports arbitrary intersection patterns, differentiable, and efficient for sparse scenes.
*   **Limitations:** Requires explicit intersection computation beforehand. Sorting is critical for correctness.
*   **When to use:** When rendering 2D Gaussians with explicit intersection pairs and needing differentiable rendering.
*   **When not to use:** When gamma-space blending is required (use [Gamma-Space Alpha Blending](7704-gamma-space-alpha-blending.md)).

## Background
Explicit Intersection Alpha Compositing is a core component of the 2D Gaussian Splatting rendering pipeline, as described in the AtomGS paper [S2, unofficial] and implemented in the `gsplat-mlx` library [S1, unofficial]. It extends the standard volume rendering equation to handle 2D Gaussian primitives projected onto the image plane. The method relies on the front-to-back compositing formula derived from classical volume rendering [S2, unofficial].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
