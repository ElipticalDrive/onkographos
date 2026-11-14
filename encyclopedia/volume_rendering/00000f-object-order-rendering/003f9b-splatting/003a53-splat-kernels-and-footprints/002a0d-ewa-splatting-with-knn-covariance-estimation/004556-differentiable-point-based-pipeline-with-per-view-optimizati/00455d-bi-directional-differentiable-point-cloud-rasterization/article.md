# Bi-Directional Differentiable Point Cloud Rasterization

Also called: Bi-directional EWA Splatting, Differentiable Point-Splatting with Bidirectional EWA, Differentiable point splatting

## Overview
Bi-directional differentiable point cloud rasterization is a method for rendering novel views from input images by projecting points into the novel view space and back using Elliptical Weighted Average (EWA) filtering [S1]. It solves the problem of optimizing per-view attributes in the 2D space of input images while maintaining high-quality novel-view synthesis, addressing ambiguity when geometry is inaccurate [S1]. It extends forward EWA point splatting by introducing a bi-directional Jacobian transformation, allowing gradients to flow from the novel view back to the input view pixels [S1].

## How it works
**Inputs:** A set of input views with calibrated cameras and associated pixel attributes, and a target novel view with its camera parameters [S1].
**Outputs:** A rendered image for the novel view and gradients for optimizing input view attributes [S1].

**Data Structures:**
*   **Point Splat:** An anisotropic 2D Gaussian $G$ associated with each reprojected pixel, defined by its mean position, covariance matrix, opacity $\alpha$, and color/attribute $c$ [S1].
*   **Splat List:** A list of splats overlapping a specific novel-view pixel, sorted by depth (front-to-back) [S1].
*   **Uncertainty $U$:** A scalar multiplicative factor applied to the covariance matrix of $G$, initialized to 0.5 [S1].

**Steps:**

1.  **Camera Selection:** Select a subset of input views $V_{sel}$ for the novel view based on pixel coverage and visibility scores [S1].
2.  **Bi-Directional Reprojection:** For each pixel $p$ in a selected input view $n$:
    a. Lift the 2D pixel coordinate to 3D space using the inverse Jacobian of the transformation from the input view to 3D [S1].
    b. Project the 3D point to the novel view using the forward Jacobian of the transformation from 3D to the novel view [S1].
    c. Compute the anisotropic 2D Gaussian $G$ for the reprojected pixel from the combined stretch of these two transformations [S1].
    d. Scale the covariance matrix of $G$ by the uncertainty factor $U$ [S1].
3.  **Splatting and Alpha Blending:** For each pixel $v$ in the novel view:
    a. Collect all splats from input views that overlap $v$ [S1].
    b. Sort the splats by depth (front-to-back) to form the set $N_n$ [S1].
    c. Perform front-to-back alpha blending to compute the final color $c_n$ [S1]:
       $$ c_n = \sum_{i \in N_n} c_{n,i} \alpha_{n,i} \prod_{j=1}^{i-1} (1 - \alpha_{n,j}) $$
       where $c_{n,i}$ and $\alpha_{n,i}$ are the color and opacity of the $i$-th splat from input view $n$ [S1].
4.  **Support Limiting:** Limit the number of splats considered per novel-view pixel to maintain efficiency and constant memory footprint [S1]:
    a. Stop accumulating if the accumulated alpha reaches 1.0 (one grayscale level) [S1].
    b. Compute eigenvalues of the covariance matrix to determine spatial decay [S1].
    c. Remove the top 3% of splats with the highest variance as outliers [S1].
    d. Calculate $\sigma_{max}$ as the highest remaining variance [S1].
    e. Define cutoff radius $r = \lfloor 3 \sigma_{max} \rfloor$ [S1].
    f. Maintain at most $k_d = 150$ front-most splat contributions per novel-view pixel [S1].
5.  **Gradient Computation:** Compute gradients for backpropagation to optimize input view attributes [S1]:
    a. Gradient with respect to color attribute $c_{n,i}$:
       $$ \frac{\partial c_n}{\partial c_{n,i}} = \alpha_{n,i} \prod_{j=1}^{i-1} (1 - \alpha_{n,j}) $$
    b. Gradient with respect to opacity $\alpha_{n,i}$:
       $$ \frac{\partial c_n}{\partial \alpha_{n,i}} = c_{n,i} \prod_{j=1}^{i-1} (1 - \alpha_{n,j}) - \sum_{l=i+1, l \neq i}^{|N_n|} c_{n,l} \alpha_{n,l} \prod_{j=1}^{l-1} (1 - \alpha_{n,j}) $$
       where the sum iterates over all splats $l$ that appear after splat $i$ in the depth-ordered list [S1]. (Note: The source notation uses $l=k+1$, which corresponds to $l=i+1$ in standard differentiable alpha blending.)

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def bi_directional_rasterize(input_views, novel_view, uncertainty_U):
    # 1. Camera Selection: Select subset of input views V_sel based on coverage/visibility [S1]
    V_sel = [v for v in input_views]

    # 2. Reproject and Splat
    novel_pixels = {v: [] for v in novel_view.pixels}

    for view_n in V_sel:
        for pixel_p in view_n.pixels:
            # Bi-directional Jacobian transformation
            # Lift 2D pixel to 3D using inverse Jacobian of input view transformation [S1]
            point_3d = lift_to_3d(pixel_p, view_n.camera, inverse_jacobian=True)
            # Project 3D point to novel view using forward Jacobian [S1]
            pixel_n, gaussian_G = project_to_novel_view(point_3d, novel_view.camera, forward_jacobian=True)
            # Apply uncertainty to covariance
            gaussian_G.covariance = [[gaussian_G.covariance[r][c] * uncertainty_U for c in range(2)] for r in range(2)]
            # Add to novel view pixel's splat list
            novel_pixels[pixel_n].append({
                'view': view_n,
                'pixel': pixel_p,
                'gaussian': gaussian_G,
                'color': pixel_p.color,
                'opacity': pixel_p.opacity,
                'depth': point_3d.z
            })

    # 3. Render and Compute Gradients
    rendered_image = {}
    gradients = {}

    for pixel_v, splat_list in novel_pixels.items():
        if not splat_list:
            continue

        # Sort splats by depth (front-to-back)
        splat_list.sort(key=lambda s: s['depth'])

        # Limit support
        splat_list = limit_splat_support(splat_list)

        # Alpha blending
        accumulated_alpha = 0.0
        color = 0.0
        splats_processed = []

        for i, splat in enumerate(splat_list):
            alpha = splat['opacity']
            c = splat['color']

            # Front-to-back compositing
            prod_term = math.prod([1.0 - s['opacity'] for s in splat_list[:i]])
            weight = alpha * prod_term
            color += weight * c
            accumulated_alpha += weight

            splats_processed.append({
                'index': i,
                'alpha': alpha,
                'color': c,
                'prod_term': prod_term
            })

            # Stop if accumulated alpha is effectively 1
            if accumulated_alpha >= 1.0 - 1e-6:
                break

        rendered_image[pixel_v] = color

        # Compute gradients for this pixel
        for splat_info in splats_processed:
            i = splat_info['index']
            alpha_i = splat_info['alpha']
            prod_term = splat_info['prod_term']
            c_i = splat_info['color']

            # Gradient w.r.t color of splat i
            grad_color = alpha_i * prod_term
            splat_info['view'].pixels[splat_info['pixel']].grad_color += grad_color

            # Gradient w.r.t opacity of splat i
            term1 = c_i * prod_term
            term2 = 0.0
            for other_splat in splats_processed:
                l = other_splat['index']
                if l > i:
                    term2 += other_splat['color'] * other_splat['alpha'] * other_splat['prod_term']

            grad_alpha = term1 - term2
            splat_info['view'].pixels[splat_info['pixel']].grad_opacity += grad_alpha

    return rendered_image, gradients

def limit_splat_support(splat_list):
    if not splat_list:
        return []

    # Remove top 3% outliers with highest variance
    variances = [s['gaussian'].variance for s in splat_list]
    sorted_vars = sorted(variances)
    threshold_idx = int(math.ceil(0.97 * len(sorted_vars)))
    threshold = sorted_vars[threshold_idx] if threshold_idx < len(sorted_vars) else sorted_vars[-1]
    filtered_splats = [s for s in splat_list if s['gaussian'].variance <= threshold]

    if not filtered_splats:
        return splat_list

    # Find max variance among remaining
    sigma_max = max([s['gaussian'].variance for s in filtered_splats])

    # Define cutoff radius
    r = int(math.floor(3 * sigma_max))

    # Filter by spatial decay (distance from center)
    final_splats = []
    for s in filtered_splats:
        if s['gaussian'].is_within_cutoff(r):
            final_splats.append(s)

    # Maintain at most k_d = 150 front-most splats
    return final_splats[:150]
```

**Parameters:**
*   **Uncertainty $U$:** Multiplicative factor on the covariance matrix of $G$. Initialized to 0.5 [S1]. Refined during optimization [S1].
*   **$k_d$:** Maximum number of splat contributions per novel-view pixel. Set to 150 [S1].
*   **Outlier Removal:** Top 3% of points with highest variance are removed [S1].
*   **Cutoff Radius:** $r = \lfloor 3 \sigma_{max} \rfloor$ [S1].

**Edge Cases and Pitfalls:**
*   **Degenerate Input:** If the novel view is the same as an input view, the camera selection algorithm is ambiguous because the first input view covers all pixels, giving equal scores to others [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Constant memory footprint per novel-view pixel due to limiting splat contributions to $k_d = 150$ [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Allows optimization of per-view features in the 2D space of input images, improving quality for difficult cases like vegetation or thin structures [S1]. Efficient filtered splatting [S1].
*   **Limitations:** Relies on accurate camera selection. Ambiguous in degenerate cases where novel view matches an input view [S1].
*   **When to use:** Free-viewpoint IBR, multiview harmonization, multi-view consistent image stylization [S1].
*   **When not to use:** Not given in the sources.

## Background
*   **Origin:** Introduced by G. Kopanas, J. Philip, T. Leimkühler & G. Drettakis in "Point-Based Neural Rendering with Per-View Optimization" [S1].
*   **History:** Extends forward EWA point splatting from Yifan et al. [YSW* 19] to be bi-directional [S1].
*   **Usage:** Free-viewpoint IBR, multiview harmonization, multi-view consistent image stylization [S1].

## Sources
[S1] Point-Based Neural Rendering with Per-View Optimization, https://www.youtube.com/watch?v=TupOewenSAk, fetched 2026-10-31
