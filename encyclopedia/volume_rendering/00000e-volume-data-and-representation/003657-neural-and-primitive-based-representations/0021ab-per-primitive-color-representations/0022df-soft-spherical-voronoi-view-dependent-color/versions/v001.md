# Soft Spherical Voronoi View-Dependent Color

Also called: SV View-Dependent Color, Soft Spherical Voronoi, Spherical Voronoi, Spherical Voronoi Representation

## Overview
Soft Spherical Voronoi (SV) is a view-dependent color model that partitions the directional domain into learnable regions with smooth boundaries, allowing a single primitive (such as a surfel or Gaussian) to represent both broad diffuse shifts and sharp specular highlights [S1, S2, unofficial]. It differs from Spherical Gaussians (SG) and Spherical Betas (SB) by using a softmax-based soft partition that ensures well-defined gradients for all sites and avoids the ill-conditioned gradients and "competition" between overlapping kernels inherent in those methods [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A viewing direction vector $\mathbf{d}$ (or $\omega$), normalized to unit length.
*   **Output:** An RGB color vector $\mathbf{c}_{out} \in \mathbb{R}^3$.

**Data Structures**
Each primitive stores $K$ sites. For each site $k \in \{1, \dots, K\}$:
1.  **Site Direction ($\mathbf{s}_k$):** A 3D unit vector ($\|\mathbf{s}_k\|=1$) defining the angular location of the Voronoi site on the sphere.
2.  **Site Color ($\mathbf{c}_k$):** A 3D vector ($\mathbb{R}^3$) representing the radiance/color associated with that site.
3.  **Temperature ($\tau_k$):** A scalar value ($\tau_k > 0$) controlling the sharpness of the partition for that site.

**Formulas**
The view-dependent color is computed as a weighted sum of the site colors:
$$ f_{SV}(\mathbf{d}) = \sum_{k=1}^K w_k(\mathbf{d}) \mathbf{c}_k $$

The weight $w_k(\mathbf{d})$ for site $k$ is computed using a softmax over the dot products of the site direction and the viewing direction, scaled by the temperature parameter:
$$ w_k(\mathbf{d}; \tau_k) = \frac{\exp(\tau_k \mathbf{s}_k \cdot \mathbf{d})}{\sum_{k'=1}^K \exp(\tau_k \mathbf{s}_{k'} \cdot \mathbf{d})} $$

*Note on Source Disagreement:* Source [S1, unofficial] defines the weight using the Euclidean distance $\|\mathbf{s}_k - \mathbf{d}\|$ in the exponent: $w_k(\mathbf{d}) = \frac{\exp(-\tau_k \|\mathbf{s}_k - \mathbf{d}\|)}{\sum_{k'=1}^K \exp(-\tau_{k'} \|\mathbf{s}_{k'} - \mathbf{d}\|)}$. Source [S2, unofficial] defines the weight using the dot product $\mathbf{s}_k \cdot \mathbf{d}$: $w_k(\omega; \tau) = \frac{\exp(\tau_k \mathbf{s}_k \cdot \omega)}{\sum_{k'=1}^{K} \exp(\tau_k \mathbf{s}_{k'} \cdot \omega)}$. Both formulations produce a soft partition where higher similarity (smaller distance or larger dot product) yields higher weight. The dot product formulation is explicitly linked to the "standard soft Spherical Voronoi model" in [S2, unofficial] and is computationally more efficient. The implementation below uses the dot product formulation [S2, unofficial].

**Parameters**
*   **$K$:** Number of sites per primitive. Not given in the sources.
*   **$\tau_k$:** Temperature parameter. Controls the sharpness of the partition. Small values produce smooth color transitions; large values approach a hard Voronoi tessellation. Typical values mentioned are 1, 5, and 25 [S2, unofficial]. If all $\tau_k$ share the same value, it is the standard model; distinct temperatures allow locally adaptive angular sharpness [S2, unofficial].

**Algorithm**
1.  Normalize the viewing direction $\mathbf{d}$ to unit length.
2.  Initialize `total_weight` = 0 and `color` = $\mathbf{0}$.
3.  For each site $k$ from 1 to $K$:
    a. Compute dot product: $dot = \mathbf{s}_k \cdot \mathbf{d}$.
    b. Compute exponent: $exp\_val = \exp(\tau_k \cdot dot)$.
    c. Add to total weight: `total_weight` += $exp\_val$.
    d. Store $exp\_val$ for this site.
4.  Initialize `color` = $\mathbf{0}$.
5.  For each site $k$ from 1 to $K$:
    a. Compute weight: $w_k = \frac{\text{stored } exp\_val_k}{\text{total\_weight}}$.
    b. Add to color: `color` += $w_k \cdot \mathbf{c}_k$.
6.  Return `color`.

**Edge Cases and Pitfalls**
*   **Numerical Stability:** When $\tau_k$ is large, $\exp(\tau_k \mathbf{s}_k \cdot \mathbf{d})$ can overflow. Standard softmax stabilization (subtracting the max exponent) should be applied [G].
*   **Zero Temperature:** $\tau_k$ must be strictly positive ($\tau_k > 0$) [S2, unofficial]. A value of 0 results in uniform weights, losing directional dependence.
*   **Performance:** Evaluation scales linearly with $K$. For large $K$ (e.g., thousands of sites used in reflection modeling), this becomes a bottleneck [S2, unofficial]. Acceleration schemes using cubemap-based partitioning to restrict the softmax to candidate sites are required in such cases [S2, unofficial].

## Subtopics

### Cubemap-Based Acceleration
An optimization for SV when $K$ is large. It partitions the unit sphere using a low-resolution cubemap and pre-assigns a fixed set of candidate sites to each texel. At runtime, the softmax is restricted to this candidate set instead of all sites, reducing complexity from $O(K)$ to $O(K_{candidate})$ [S2, unofficial].

### SV for Diffuse Appearance
Used in radiance fields (e.g., 3D Gaussian Splatting) to fit diffuse appearance. It achieves competitive results with simpler optimization compared to SH or SG [S2, unofficial].

### SV for Reflections
Used as learnable reflection probes, taking reflected directions as input. This addresses the failure of Spherical Harmonics to capture specular reflections [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(K)$ per primitive per ray, where $K$ is the number of sites. With cubemap acceleration, it is $O(K_{candidate})$ where $K_{candidate} \ll K$ [S2, unofficial].
*   **Memory Complexity:** $O(K \cdot D)$ per primitive, where $D$ is the dimensionality of the color (3 for RGB) plus the site directions and temperatures. Specifically, $3K$ (sites) + $3K$ (colors) + $K$ (temperatures) = $7K$ floats per primitive [S2, unofficial].
*   **Strengths:**
    *   Avoids ill-conditioned gradients and sensitivity to initialization found in SG and SB [S2, unofficial].
    *   Provides a stable parameterization for view-dependent effects [S2, unofficial].
    *   Can model both smooth diffuse and sharp specular features [S1, S2, unofficial].
*   **Limitations:**
    *   Linear scaling with $K$ makes naive evaluation expensive for high-fidelity reflections requiring many sites [S2, unofficial].
    *   Requires careful tuning of $\tau_k$ to balance smoothness and sharpness [S2, unofficial].
*   **When to Use:** When view-dependent color is needed and optimization stability is a concern, or when modeling reflections where SH fails. Use cubemap acceleration if $K$ is large.

## Background
Soft Spherical Voronoi was introduced as a unified framework for appearance representation in 3D Gaussian Splatting [S2, unofficial]. It was proposed to address the limitations of Spherical Harmonics (SH), Spherical Gaussians (SG), and Spherical Betas (SB), which suffer from Gibbs ringing, ill-conditioned gradients, or optimization complexity [S2, unofficial]. The method partitions the directional domain into learnable regions, providing an intuitive and stable parameterization [S2, unofficial]. It has been applied to fitting radiance fields for diffuse appearance and modeling reflections as learnable reflection probes [S2, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
[S2] Spherical Voronoi: Directional Appearance as a Differentiable Partition of the Sphere, https://doi.org/10.48550/arXiv.2512.14180, fetched 2026-10-30 (unofficial)
