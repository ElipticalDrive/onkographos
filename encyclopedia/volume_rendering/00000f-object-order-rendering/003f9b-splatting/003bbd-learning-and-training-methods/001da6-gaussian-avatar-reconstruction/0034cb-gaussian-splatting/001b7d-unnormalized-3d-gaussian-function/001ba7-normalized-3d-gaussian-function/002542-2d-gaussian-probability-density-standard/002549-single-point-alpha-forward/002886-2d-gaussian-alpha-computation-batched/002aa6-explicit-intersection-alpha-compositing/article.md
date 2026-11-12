# Explicit Intersection Alpha Compositing

Also called: accumulate

## Overview
Explicit Intersection Alpha Compositing converts a list of Gaussian-pixel intersection records into a final RGB image and an alpha mask by performing front-to-back alpha compositing using explicit intersection pairs (gaussian_id, pixel_id, image_id) [S1, unofficial]. It aggregates contributions from multiple overlapping Gaussians onto each pixel, relying on automatic differentiation for the backward pass [S1, unofficial]. Unlike its parent method, [2D Gaussian Alpha Computation (Batched)](4273-2d-gaussian-alpha-computation-batched.md), which computes individual alpha values, this method manages ray indices and transmittance to accumulate final pixel colors [S1, unofficial]. It is distinguished from [Gamma-Space Alpha Blending](7704-gamma-space-alpha-blending.md) by handling arbitrary intersection patterns via a flat list of pairs rather than assuming a fixed accumulation order within a single color space context [S1, unofficial].

## How it works
**Inputs:**
- `gaussian_id`: Array of shape $[M]$ containing the index of the Gaussian for each intersection [S1, unofficial].
- `pixel_id`: Array of shape $[M]$ containing the linearized pixel index for each intersection [S1, unofficial].
- `image_id`: Array of shape $[M]$ containing the batch image index for each intersection [S1, unofficial].
- `means2d_selected`: Array of shape $[M, 2]$ containing the 2D mean positions of the Gaussians at the intersection points [S1, unofficial].
- `conics_selected`: Array of shape $[M, 3]$ containing the inverse covariance matrix elements (upper triangle) for the Gaussians [S1, unofficial].
- `opacities_selected`: Array of shape $[M]$ containing the scalar opacity of each Gaussian [S1, unofficial].
- `colors_selected`: Array of shape $[M, C]$ containing the color vectors for each Gaussian [S1, unofficial].
- `image_height`, `image_width`: Integers defining the resolution of the output image [S1, unofficial].
- `MAX_ALPHA`: Float constant for alpha clamping (typically $< 1.0$ to prevent numerical instability) [S1, unofficial].

**Outputs:**
- `renders`: Array of shape $[H, W, C]$ containing the final composited RGB colors [S1, unofficial].
- `alphas_out`: Array of shape $[H, W, 1]$ containing the final composited alpha values [S1, unofficial].

**Data Structures:**
- `pixel_coords`: Array of shape $[M, 2]$ storing the floating-point pixel coordinates [S1, unofficial].
- `sigmas`: Array of shape $[M]$ storing the Gaussian exponent values [S1, unofficial].
- `alphas_per_isect`: Array of shape $[M]$ storing the clamped alpha for each intersection [S1, unofficial].
- `ray_indices`: Array of shape $[M]$ storing the unique ray identifier for each intersection [S1, unofficial].
- `weights`: Array of shape $[M]$ storing the front-to-back compositing weights [S2, unofficial].
- `_transmittances`: Array of shape $[M]$ storing the transmittance values (unused in final output but computed for correctness) [S1, unofficial].

**Steps:**

1. **Compute Pixel Coordinates:** Convert linear pixel IDs to 2D coordinates [S1, unofficial]. The coordinates are centered such that the center of pixel $(x, y)$ is at $(x+0.5, y+0.5)$ [G].
   $$ \text{pixel\_coords} = [\text{pixel\_ids}_x, \text{pixel\_ids}_y] + 0.5 $$
   where `pixel_ids_x` and `pixel_ids_y` are derived from `pixel_id` using integer division and modulo operations [G].

2. **Gather Gaussian Parameters:** Using `image_id` and `gaussian_id`, gather the corresponding `means2d`, `conics`, `opacities`, and `colors` from the global Gaussian attributes [S1, unofficial]. This results in the selected arrays of shape $[M, \cdot]$ [G].

3. **Compute Sigma (Gaussian Exponent):** Calculate the exponent value $\sigma$ for each intersection using the conic matrix and the delta vector (distance from the Gaussian mean to the pixel center) [S1, unofficial].
   Let $\Delta = [\Delta_x, \Delta_y]$ be the vector from `means2d_selected` to `pixel_coords` [G].
   Let the conic matrix $C$ be represented by elements $C_{00}, C_{01}, C_{11}$ from `conics_selected` (where $C_{01}$ is the off-diagonal term) [G].
   $$ \sigma = 0.5 \left( C_{00} \Delta_x^2 + C_{11} \Delta_y^2 \right) + C_{01} \Delta_x \Delta_y $$
   In code:
   ```python
   deltas = pixel_coords - means2d_selected
   sigmas = 0.5 * (conics_selected[:, 0] * deltas[:, 0]**2 + 
                   conics_selected[:, 2] * deltas[:, 1]**2) + \
            conics_selected[:, 1] * deltas[:, 0] * deltas[:, 1]
   ```

4. **Compute Per-Intersection Alpha:** Calculate the alpha value for each intersection by combining the Gaussian opacity with the exponential decay of the sigma value, then clamp it to `MAX_ALPHA` [S1, unofficial].
   $$ \alpha_i = \min(\text{opacity}_i \cdot e^{-\sigma_i}, \text{MAX\_ALPHA}) $$
   ```python
   alphas_per_isect = minimum(opacities_selected * exp(-sigmas), MAX_ALPHA)
   ```

5. **Compute Ray Indices:** Map each intersection to a unique ray index to handle batched rendering [S1, unofficial].
   $$ \text{ray\_index}_i = \text{image\_id}_i \cdot (H \cdot W) + \text{pixel\_id}_i $$
   ```python
   ray_indices = image_id * (image_height * image_width) + pixel_id
   ```

6. **Compute Compositing Weights:** Perform front-to-back alpha compositing to determine the weight of each intersection [S2, unofficial]. This step requires sorting intersections by depth (distance from camera) for each ray, as the viewing transformation generates a sorted list $N := \{G_1, ..., G_N\}$ [S2, unofficial]. The weight formula is $w_i(x) = \alpha_i G_i(x) \prod_{j=1}^{i-1} (1 - \alpha_j G_j(x))$ [S2, unofficial].
   ```python
   weights = zeros_like(alphas_per_isect)
   transmittances = zeros_like(alphas_per_isect)
   ray_transmittance = ones(total_pixels)
   for i in range(len(alphas_per_isect)):
       r = ray_indices[i]
       weights[i] = alphas_per_isect[i] * ray_transmittance[r]
       transmittances[i] = ray_transmittance[r]
       ray_transmittance[r] *= (1.0 - alphas_per_isect[i])
   ```

7. **Accumulate Colors:** Accumulate the weighted colors for each ray [S1, unofficial].
   $$ C_{\text{out}} = \sum_{i} w_i \cdot \text{color}_i $$
   ```python
   renders_flat = zeros((total_pixels, colors_selected.shape[1]))
   for i in range(len(alphas_per_isect)):
       r = ray_indices[i]
       renders_flat[r] += weights[i] * colors_selected[i]
   ```

8. **Accumulate Alpha:** Accumulate the weights (which represent the total alpha contribution) for each ray [S1, unofficial].
   $$ \alpha_{\text{out}} = \sum_{i} w_i $$
   ```python
   alphas_out_flat = zeros(total_pixels)
   for i in range(len(alphas_per_isect)):
       r = ray_indices[i]
       alphas_out_flat[r] += weights[i]
   ```

9. **Reshape Outputs:** Reshape the flat arrays `renders` and `alphas_out` back into image dimensions [S1, unofficial].
   - `renders`: $[H, W, C]$
   - `alphas_out`: $[H, W, 1]$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def explicit_intersection_alpha_compositing(
    gaussian_id, pixel_id, image_id,
    means2d, conics, opacities, colors,
    image_height, image_width, MAX_ALPHA=0.99
):
    # 1. Compute pixel coordinates
    pixel_ids_x = pixel_id // image_width
    pixel_ids_y = pixel_id % image_width
    pixel_coords = stack([pixel_ids_x, pixel_ids_y], axis=-1).astype(float32) + 0.5

    # 2. Gather parameters
    means2d_sel = means2d[gaussian_id]
    conics_sel = conics[gaussian_id]
    opacities_sel = opacities[gaussian_id]
    colors_sel = colors[gaussian_id]

    # 3. Compute sigma
    deltas = pixel_coords - means2d_sel
    sigmas = 0.5 * (conics_sel[:, 0] * deltas[:, 0]**2 + 
                    conics_sel[:, 2] * deltas[:, 1]**2) + \
             conics_sel[:, 1] * deltas[:, 0] * deltas[:, 1]

    # 4. Compute alpha
    alphas = minimum(opacities_sel * exp(-sigmas), MAX_ALPHA)

    # 5. Compute ray indices
    total_pixels = image_height * image_width
    ray_indices = image_id * total_pixels + pixel_id

    # 6. Compute compositing weights (front-to-back)
    # Requires intersections sorted by depth per ray [S2, unofficial]
    weights = zeros_like(alphas)
    transmittances = zeros_like(alphas)
    ray_transmittance = ones(total_pixels)
    for i in range(len(alphas)):
        r = ray_indices[i]
        weights[i] = alphas[i] * ray_transmittance[r]
        transmittances[i] = ray_transmittance[r]
        ray_transmittance[r] *= (1.0 - alphas[i])

    # 7. Accumulate colors
    renders_flat = zeros((total_pixels, colors_sel.shape[1]))
    for i in range(len(alphas)):
        r = ray_indices[i]
        renders_flat[r] += weights[i] * colors_sel[i]

    # 8. Accumulate alpha
    alphas_out_flat = zeros(total_pixels)
    for i in range(len(alphas)):
        r = ray_indices[i]
        alphas_out_flat[r] += weights[i]

    # 9. Reshape outputs
    renders = reshape(renders_flat, (image_height, image_width, colors_sel.shape[1]))
    alphas_out = reshape(alphas_out_flat, (image_height, image_width, 1))

    return renders, alphas_out
```

**Edge Cases and Pitfalls:**
- **Sorting:** The correctness of front-to-back compositing relies on intersections being sorted by depth for each ray [S2, unofficial]. If the input list is not sorted, the result will be incorrect [G].
- **Alpha Clamping:** `MAX_ALPHA` is used to prevent numerical instability during the backward pass [S1, unofficial]. A value too close to 1.0 can cause vanishing gradients [G].
- **Precision:** The computation of `sigmas` involves squared terms and products, which can lead to precision issues for very large or very small Gaussians [G]. Using `float32` is standard, but `float64` may be needed for extreme cases [G].

## Subtopics

### [Gamma-Space Alpha Blending](7704-gamma-space-alpha-blending.md)
Gamma-Space Alpha Blending performs Gaussian alpha compositing directly in the Gamma color space to match the training conditions of standard 3D Gaussian Splatting (3DGS) assets. It solves the quality degradation caused by converting colors to linear space before blending, which is the default behavior in modern linear-rendering pipelines. Unlike its parent method, Explicit Intersection Alpha Compositing, which handles arbitrary intersection patterns via explicit pairs, this method assumes a standard front-to-back or back-to-front accumulation where the entire splatting pass occurs within a single Gamma-space context.

## Operating characteristics
- **Time Complexity:** $O(M)$ where $M$ is the number of intersections [G]. The dominant operations are the gather, sigma computation, and the compositing accumulation, all of which are linear in the number of intersections [G].
- **Memory Complexity:** $O(M)$ for storing the intermediate arrays (`sigmas`, `alphas_per_isect`, `weights`, etc.) [G]. The output size is $O(H \cdot W \cdot C)$ [G].
- **Performance:** The method is highly parallelizable on GPUs due to the independent computation of sigma and alpha per intersection, and the efficient accumulation using atomic operations or parallel reduction [G].
- **Strengths:** Explicit handling of intersections allows for flexible rendering pipelines and easy integration with automatic differentiation frameworks [S1, unofficial].
- **Limitations:** Requires sorting of intersections by depth, which can be a bottleneck if not handled efficiently [S2, unofficial]. The clamping of alpha introduces a small approximation error [G].

## Background
This method is part of the 2D Gaussian Splatting (2DGS) rendering pipeline, specifically the compositing stage [S1, unofficial]. It builds on [2D Gaussian Alpha Computation (Batched)](4273-2d-gaussian-alpha-computation-batched.md) and is implemented in libraries such as `gsplat-mlx` [S1, unofficial]. The approach of explicit intersection compositing is common in differentiable rendering to allow gradients to flow through the alpha values and colors [S1, unofficial].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
