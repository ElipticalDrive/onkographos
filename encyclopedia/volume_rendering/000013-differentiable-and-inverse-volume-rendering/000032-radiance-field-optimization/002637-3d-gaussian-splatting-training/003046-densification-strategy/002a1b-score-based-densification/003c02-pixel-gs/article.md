# Pixel-GS

Also called: Pixel-based densification, Projected area gradient scaling

## Overview
Pixel-GS is a densification strategy for 3D Gaussian Splatting that prioritizes splitting Gaussians based on their spatial impact on the rendered image. It solves the problem of inefficient primitive distribution by scaling the densification gradient by the number of pixels a Gaussian influences, ensuring that primitives causing errors across many pixels are split first [S2, unofficial]. Unlike standard 3DGS which uses average positional gradients, Pixel-GS accounts for the projected area and depth, yielding state-of-the-art rendering quality at the cost of a higher total number of Gaussian primitives [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The set of 3D Gaussians $\{G_i\}$ from the previous training iteration, their positional gradients $\nabla_{pos} G_i$, and the camera parameters for all input viewpoints.
*   **Output:** A modified set of Gaussians where high-impact primitives are split or cloned, and low-impact primitives are retained.

**Data Structures:**
Each Gaussian $G_i$ maintains:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$
*   Covariance $\Sigma_i$ (or rotation $\mathbf{q}_i$ and scale $\mathbf{s}_i$)
*   Opacity $\alpha_i$
*   Color coefficients
*   Accumulated gradient magnitude $g_i$ (initialized to 0)
*   Pixel coverage count $c_i$ (initialized to 0)

**Steps:**
1.  **Forward Pass:** Render the scene from all input viewpoints. For each Gaussian $G_i$ contributing to the image, determine the set of pixels $P_i$ it influences.
2.  **Pixel Counting:** For each Gaussian $G_i$, count the number of pixels it covers across all viewpoints:
    $$ c_i = \sum_{v \in \text{views}} |P_i^{(v)}| $$
    [S1]
3.  **Gradient Scaling:** Compute the standard positional gradient magnitude $||\nabla_{pos} G_i||$. Scale this gradient by the pixel coverage count $c_i$ and a depth-dependent factor $d_i$ to suppress floaters in the background:
    $$ \tilde{g}_i = c_i \cdot ||\nabla_{pos} G_i|| \cdot d_i $$
    The depth factor $d_i$ typically increases with distance to the camera to encourage densification in background regions where Gaussians are larger and sparser [S1].
4.  **Densification Decision:** Compare $\tilde{g}_i$ against a threshold $\tau$. If $\tilde{g}_i > \tau$, mark $G_i$ for splitting or cloning.
5.  **Splitting:** For marked Gaussians, create two new Gaussians by perturbing the position along the principal axes of the covariance matrix, halving the scale, and copying other attributes. Reset $g_i$ and $c_i$ for the new primitives.
6.  **Update:** Accumulate $\tilde{g}_i$ into the Gaussian's history for the next iteration.

**Parameters:**
*   Threshold $\tau$: Not given in the sources.
*   Depth factor $d_i$: Not given in the sources.
*   Number of viewpoints: All input views [S1].

**Edge Cases:**
*   **Precision:** High pixel counts for large foreground Gaussians may dominate the gradient, potentially starving background refinement if $d_i$ is not tuned [S1].
*   **Boundaries:** Gaussians at the edge of the view frustum may have artificially low pixel counts; culling should be handled before counting.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pixel_gs_densification(gaussians, cameras, threshold):
    # Reset counters
    for g in gaussians:
        g.pixel_count = 0
        g.accumulated_grad = 0.0

    # Render and count pixels for all views
    for cam in cameras:
        rendered_image, depth_map = render(gaussians, cam)
        # Identify which Gaussian contributes to which pixel
        # This requires a visibility buffer or rasterization metadata
        pixel_contributions = get_pixel_contributions(rendered_image, gaussians)
        
        for g in gaussians:
            # Count pixels influenced by this Gaussian in this view
            count = count_pixels_influenced(g, pixel_contributions)
            g.pixel_count += count

    # Compute scaled gradients
    for g in gaussians:
        # Standard positional gradient from backprop
        pos_grad_mag = compute_gradient_magnitude(g.position_gradient)
        
        # Depth factor: suppress floaters (background) or encourage them?
        # Sources say "scales the gradient according to the distance to the camera to suppress floaters"
        # This implies distance factor reduces gradient for far objects? 
        # Or increases it to densify background? 
        # "suppress floaters" usually means preventing splitting of noisy background noise.
        # However, "encourage densification in background regions" is also stated.
        # We assume a factor f(dist) that balances these.
        dist = compute_distance_to_camera(g.position, cam)
        depth_factor = compute_depth_factor(dist)
        
        scaled_grad = g.pixel_count * pos_grad_mag * depth_factor
        g.accumulated_grad += scaled_grad

    # Split Gaussians exceeding threshold
    new_gaussians = []
    for g in gaussians:
        if g.accumulated_grad > threshold:
            # Split logic
            g1, g2 = split_gaussian(g)
            new_gaussians.append(g1)
            new_gaussians.append(g2)
        else:
            new_gaussians.append(g)
            
    return new_gaussians
```

## Subtopics

### [PixelGS Densification Criteria](9689-pixelgs-densification-criteria.md)
Modifies the gradient strategy to account for spatial footprint and applies depth-dependent scaling to suppress floaters and encourage background densification.

## Operating characteristics
*   **Time Complexity:** Higher than standard 3DGS due to the need to track pixel contributions per Gaussian across all views.
*   **Memory Complexity:** Higher, as it requires storing pixel coverage counts and potentially larger metadata for gradient accumulation.
*   **Performance:** Yields state-of-the-art rendering quality compared to standard 3DGS [S1].
*   **Limitations:** Typically results in a higher number of Gaussian primitives, increasing memory usage and potentially slowing down rendering [S1].
*   **When to use:** When reconstruction quality is prioritized over model size and training speed.
*   **When not to use:** When memory or rendering speed is constrained.

## Background
Pixel-GS was introduced as a variant of 3D Gaussian Splatting to improve densification efficiency. It is cited in recent surveys of 3D Gaussian Splatting advances [S1] and compared with other methods like GaussianPro and AbsGS [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
