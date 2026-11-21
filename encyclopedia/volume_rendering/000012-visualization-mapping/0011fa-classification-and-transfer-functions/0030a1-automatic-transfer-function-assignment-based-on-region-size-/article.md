# Automatic Transfer Function Assignment based on Region Size and Occlusion

Also called: Occlusion-aware TF Assignment, Region-based TF Assignment

## Overview
This method automatically assigns visual parameters (opacity and color) to segmented volume regions (clusters) based on their geometric size and relative spatial distance to ensure the visibility of smaller or interior structures that might otherwise be occluded by larger ones [S1, unofficial]. It solves the problem of manual fine-tuning in direct volume rendering by making large structures more transparent and small structures more opaque, thereby revealing hidden details [S1, unofficial]. It differs from its parent method, [Automatic and Data-Driven Generation](7609-automatic-and-data-driven-generation.md), by using specific geometric heuristics (size and occlusion) rather than general data-driven optimization, and from [Automated Visibility-Based TF Parameter Optimization](7072-automated-visibility-based-tf-parameter-optimization.md) by relying on static geometric properties rather than enforcing quantitative visibility thresholds for prioritized structures [S1, unofficial].

## How it works
**Inputs:**
*   A set of segmented regions (clusters) $R_i$, where each region contains a set of voxels $V_i$ [S1, unofficial].
*   For each voxel $v_j = (v_j^x, v_j^y, v_j^z)$, its position and intensity value $f_v$ [S1, unofficial].
*   Gradient magnitude $\|\nabla v\|$ for each voxel (computed from the volume data) [S1, unofficial].
*   Parameters: $k_d \ge 1$ (occlusion threshold factor, typically 1), $k_s$ (opacity scaling factor, e.g., 0.1 or 0.3), $\alpha_{min}, \alpha_{max}$ (opacity range), and a color mapping strategy (cold-to-hot spectrum or pre-defined array) [S1, unofficial].

**Outputs:**
*   For each region $R_i$, a base color $c_i$ and a base opacity $\alpha_i$ [S1, unofficial].
*   For each voxel $v$ in region $R_i$, a final voxel opacity $\alpha_v^i$ and voxel color $c_v^i$ [S1, unofficial].

**Data Structures:**
*   **Region Statistics:** For each region $R_i$, store:
    *   $N_i$: Number of voxels.
    *   $\mu_i = (\mu_i^x, \mu_i^y, \mu_i^z)$: Mean position of voxels in $R_i$.
    *   $\sigma_i$: Standard deviation of voxel positions (region size).
    *   $S_i$: Number of regions occluded by $R_i$.
    *   $max\_grad_i$: Maximum gradient magnitude in $R_i$.
    *   $max\_int_i$: Maximum intensity value in $R_i$.

**Steps:**

1.  **Compute Region Statistics:**
    For each region $R_i$:
    $$ \mu_i = \frac{1}{N_i} \sum_{v_j \in R_i} v_j \quad (4) $$
    $$ \sigma_i = \sqrt{\frac{1}{N_i} \sum_{v_j \in R_i} (v_j - \mu_i)^2} \quad (3) $$
    $$ max\_grad_i = \max_{u \in R_i} \|\nabla u\| $$
    $$ max\_int_i = \max_{u \in R_i} f_u \quad [S1, unofficial] $$

2.  **Determine Occlusion Relationships:**
    For every pair of regions $(R_i, R_j)$:
    Calculate Euclidean distance between means:
    $$ D(R_i, R_j) = \sqrt{(\mu_i^x - \mu_j^x)^2 + (\mu_i^y - \mu_j^y)^2 + (\mu_i^z - \mu_j^z)^2} \quad (5) $$
    Region $R_i$ occludes $R_j$ if:
    $$ \sigma_i > \sigma_j \quad \text{AND} \quad \sigma_i > k_d \cdot D(R_i, R_j) \quad (6) $$
    Count $S_i$, the number of regions $R_j$ occluded by $R_i$ [S1, unofficial].

3.  **Assign Base Opacity ($\alpha_i$):**
    Map $\sigma_i$ linearly to $[\alpha_{min}, \alpha_{max}]$ to get $\alpha_i^*$:
    $$ \alpha_i^* = \frac{\max_j \sigma_j - \sigma_i}{\max_j \sigma_j - \min_j \sigma_j} (\alpha_{max} - \alpha_{min}) + \alpha_{min} \quad (8) $$
    Apply scaling based on occlusion count:
    $$ \alpha_i = \frac{\alpha_i^*}{k_s (S_i + 1)} \quad (7) $$
    *Note: This makes larger regions (higher $\sigma_i$) more transparent and regions occluding many others ($S_i$ high) more transparent* [S1, unofficial].

4.  **Assign Base Color ($c_i$):**
    Map the ratio $\frac{\sigma_i}{\max_j \sigma_j}$ to a color spectrum.
    *   Small regions ($\sigma_i$ low) $\rightarrow$ Hot colors (e.g., red/yellow).
    *   Large regions ($\sigma_i$ high) $\rightarrow$ Cool colors (e.g., blue/cyan).
    *   Alternatively, use a pre-defined color array indexed by region ID or rank [S1, unofficial].

5.  **Compute Voxel-Level Parameters:**
    For each voxel $v$ in region $R_i$:
    *   **Voxel Opacity:** Modulate base opacity by gradient ratio to enhance boundaries:
        $$ \alpha_v^i = \alpha_i \cdot \frac{\|\nabla v\|}{max\_grad_i} \quad (9) $$
    *   **Voxel Color:** Scale base color by intensity ratio, applied only to the **Brightness** component in HSV space:
        $$ c_v^i = c_i \cdot \frac{f_v}{max\_int_i} \quad (10) $$
        Specifically, if $c_i = (H, S, V)$, then $c_v^i = (H, S, V \cdot \frac{f_v}{max\_int_i})$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def norm(v):
    return (v[0]**2 + v[1]**2 + v[2]**2)**0.5

def lerp(a, b, t):
    return (a[0] + t*(b[0]-a[0]), a[1] + t*(b[1]-a[1]), a[2] + t*(b[2]-a[2]))

def rgb_to_hsv(c):
    r, g, b = c
    mx, mn = max(r, g, b), min(r, g, b)
    d = mx - mn
    h = 0.0
    if d > 0:
        if mx == r: h = 60 * (((g - b) / d) % 6)
        elif mx == g: h = 60 * (((b - r) / d) + 2)
        else: h = 60 * (((r - g) / d) + 4)
    s = mx / mx if mx > 0 else 0.0
    return (h, s, mx)

def hsv_to_rgb(h, s, v):
    c = v * s
    x = c * (1 - abs((h / 60) % 2 - 1))
    m = v - c
    if h < 60: r, g, b = c, x, 0
    elif h < 120: r, g, b = x, c, 0
    elif h < 180: r, g, b = 0, c, x
    elif h < 240: r, g, b = 0, x, c
    elif h < 300: r, g, b = x, 0, c
    else: r, g, b = c, 0, x
    return (r + m, g + m, b + m)

def compute_region_stats(regions):
    stats = {}
    for i, region in enumerate(regions):
        voxels = region.voxels
        N = len(voxels)
        mu = sum(v.pos for v in voxels) / N
        sigma = (sum((v.pos - mu)**2 for v in voxels) / N)**0.5
        max_grad = max(v.gradient for v in voxels)
        max_int = max(v.intensity for v in voxels)
        stats[i] = {'N': N, 'mu': mu, 'sigma': sigma, 'max_grad': max_grad, 'max_int': max_int}
    return stats

def determine_occlusions(stats, k_d=1.0):
    region_ids = list(stats.keys())
    occlusion_counts = {i: 0 for i in region_ids}
    for i in region_ids:
        for j in region_ids:
            if i == j: continue
            sigma_i = stats[i]['sigma']
            sigma_j = stats[j]['sigma']
            dist = norm((stats[i]['mu'][0]-stats[j]['mu'][0], stats[i]['mu'][1]-stats[j]['mu'][1], stats[i]['mu'][2]-stats[j]['mu'][2]))
            if sigma_i > sigma_j and sigma_i > k_d * dist:
                occlusion_counts[i] += 1
    return occlusion_counts

def assign_visual_params(stats, occlusion_counts, k_s=0.1, alpha_min=0.0, alpha_max=1.0):
    sigmas = [s['sigma'] for s in stats.values()]
    min_sigma = min(sigmas)
    max_sigma = max(sigmas)
    params = {}
    for i, s in stats.items():
        sigma = s['sigma']
        if max_sigma == min_sigma:
            alpha_star = alpha_max
        else:
            alpha_star = ((max_sigma - sigma) / (max_sigma - min_sigma)) * (alpha_max - alpha_min) + alpha_min
        S_i = occlusion_counts[i]
        alpha_i = alpha_star / (k_s * (S_i + 1))
        size_ratio = sigma / max_sigma
        color_i = lerp((1.0, 0.0, 0.0), (0.0, 0.0, 1.0), size_ratio)
        params[i] = {'alpha': alpha_i, 'color': color_i}
    return params

def render_voxels(regions, params):
    for i, region in enumerate(regions):
        p = params[i]
        alpha_base = p['alpha']
        color_base = p['color']
        max_grad = region.stats['max_grad']
        max_int = region.stats['max_int']
        for v in region.voxels:
            grad_ratio = v.gradient / max_grad if max_grad > 0 else 0
            alpha_v = alpha_base * grad_ratio
            h, s, v_hsv = rgb_to_hsv(color_base)
            intensity_ratio = v.intensity / max_int if max_int > 0 else 0
            v_new = v_hsv * intensity_ratio
            color_v = hsv_to_rgb(h, s, v_new)
            yield v.pos, alpha_v, color_v
```

## Subtopics

### [Automated Visibility-Based TF Parameter Optimization](7072-automated-visibility-based-tf-parameter-optimization.md)
Adjusts TF parameters to enforce quantitative visibility thresholds for Structures of Interest, rather than relying on geometric size/occlusion heuristics.

## Operating characteristics
*   **Time Complexity:** Computing region statistics is $O(N)$ where $N$ is total voxels [G]. Determining occlusions is $O(K^2)$ where $K$ is the number of regions [G]. Rendering is $O(N)$ [G].
*   **Memory Complexity:** $O(N)$ to store voxel data and region membership. $O(K)$ for region statistics [G].
*   **Performance:** In experiments on 16-bit CT volumes (e.g., 256×256×161), pre-processing took 151–215 seconds on a 2.66 GHz Intel i5-750 CPU [S1, unofficial]. Real-time frame rates were achieved on a GPU-based ray marching renderer at 512×512 resolution [S1, unofficial].
*   **Strengths:** Automatically reveals small, interior structures occluded by large ones without manual intervention [S1, unofficial]. Enhances boundary visibility via gradient modulation [S1, unofficial].
*   **Limitations:** Relies on accurate segmentation/clustering. Color assignment is heuristic (size-based) and may not reflect semantic material properties [S1, unofficial]. The occlusion model is simplified (based on mean distance and size, not ray-casting depth) [S1, unofficial].
*   **When to use:** When the goal is to visualize internal structures of complex, multi-scale volumes where manual TF tuning is difficult [S1, unofficial].
*   **When not to use:** When semantic accuracy of color is critical (requires external knowledge) or when regions are not well-separated by size/position [S1, unofficial].

## Background
This method was proposed in the context of automatic transfer function design using clustering on LH (Local Histogram) space [S1, unofficial]. It addresses the difficulty of assigning visual parameters to clusters derived from volume data, particularly when significant overlap exists between clusters [S1, unofficial]. The approach leverages the intuition that smaller structures are more likely to be occluded and thus require higher opacity, while larger structures should be more transparent to reveal interiors [S1, unofficial].

## Sources
[S1] Automatic Transfer Function Design for Volumetric Data Visualization using Clustering on LH Space, https://www.researchgate.net/publication/265318052_Automatic_Transfer_Function_Design_for_Volumetric_Data_Visualization_using_Clustering_on_LH_Space, fetched 2026-10-28 (unofficial)
