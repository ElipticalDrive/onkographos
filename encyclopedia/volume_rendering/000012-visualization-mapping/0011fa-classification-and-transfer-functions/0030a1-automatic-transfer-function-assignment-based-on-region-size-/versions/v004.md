# Automatic Transfer Function Assignment based on Region Size and Occlusion

Also called: Occlusion-aware TF Assignment, Region-based TF Assignment

## Overview
This method automatically assigns opacity and color parameters to segmented volumetric regions (clusters) to ensure the visibility of smaller or interior structures that might otherwise be hidden by larger, opaque neighbors [S1, unofficial]. It differs from sibling methods like [Histogram-Based Transfer Function Generation](6182-histogram-based-transfer-function-generation.md) by operating on spatial geometric properties (region size and occlusion relationships) rather than scalar intensity distributions, and from [Valley Cell-Based Clustering Transfer Function Design](7418-valley-cell-based-clustering-transfer-function-design.md) by explicitly calculating inter-region occlusion to modulate transparency [S1, unofficial].

## How it works
**Inputs:**
*   A set of segmented regions $R_i$, where each region $R_i$ consists of a set of voxels $V_i$.
*   For each voxel $v_j \in R_i$, its 3D position $\mathbf{v}_j = (v_{jx}, v_{jy}, v_{jz})$ and its scalar intensity value $f_v$.
*   Gradient magnitude $\|\nabla v\|$ for each voxel (computed from the volume data).
*   Parameters: $k_d \ge 1$ (occlusion threshold factor, typically 1), $k_s$ (opacity scaling factor, e.g., 0.1 or 0.3), $\alpha_{min}, \alpha_{max}$ (opacity range), and a color mapping strategy (cold-to-hot spectrum or pre-defined array).

**Outputs:**
*   For each voxel $v$ in region $R_i$: an opacity $\alpha^i_v$ and a color $c^i_v$.

**Data Structures:**
*   Regions are stored as lists of voxel indices or coordinates.
*   Pre-computed statistics for each region $R_i$:
    *   $N_i$: Number of voxels.
    *   $\mu_i$: Mean position vector $(\mu_{ix}, \mu_{iy}, \mu_{iz})$.
    *   $\sigma_i$: Standard deviation of voxel positions (region size proxy).
    *   $S_i$: Count of regions occluded by $R_i$.
    *   $F_{max, i}$: Maximum intensity value in $R_i$.
    *   $G_{max, i}$: Maximum gradient magnitude in $R_i$.

**Steps:**

1.  **Compute Region Statistics:**
    For each region $R_i$:
    *   Calculate mean position $\mu_i$:
        $$ \mu_i = \frac{1}{N_i} \sum_{v_j \in R_i} \mathbf{v}_j $$
    *   Calculate size proxy $\sigma_i$ (standard deviation of positions):
        $$ \sigma_i = \sqrt{\frac{1}{N_i} \sum_{v_j \in R_i} \|\mathbf{v}_j - \mu_i\|^2} $$
    *   Find $F_{max, i} = \max_{u \in R_i} f_u$ and $G_{max, i} = \max_{u \in R_i} \|\nabla u\|$.

2.  **Calculate Inter-Region Distances:**
    For every pair of regions $(R_i, R_j)$, compute Euclidean distance between their means:
    $$ D(R_i, R_j) = \sqrt{(\mu_{ix} - \mu_{jx})^2 + (\mu_{iy} - \mu_{jy})^2 + (\mu_{iz} - \mu_{jz})^2} $$

3.  **Determine Occlusion Relationships:**
    Region $R_i$ is considered to occlude region $R_j$ if both conditions are met:
    $$ \sigma_i > \sigma_j \quad \text{AND} \quad \sigma_i > k_d \cdot D(R_i, R_j) $$
    Count $S_i$, the number of regions $R_j$ occluded by $R_i$.

4.  **Assign Base Opacity $\alpha^*_i$:**
    Map the region size $\sigma_i$ linearly to the target opacity range $[\alpha_{min}, \alpha_{max}]$. Smaller regions (smaller $\sigma_i$) receive higher opacity:
    $$ \alpha^*_i = \frac{\max_j(\sigma_j) - \sigma_i}{\max_j(\sigma_j) - \min_j(\sigma_j)} (\alpha_{max} - \alpha_{min}) + \alpha_{min} $$

5.  **Calculate Final Region Opacity $\alpha_i$:**
    Adjust $\alpha^*_i$ based on the number of occluded regions $S_i$ and the scaling factor $k_s$:
    $$ \alpha_i = \frac{\alpha^*_i}{k_s (S_i + 1)} $$
    *Note: This reduces opacity for large regions that occlude many others, enhancing visibility of internal structures [S1, unofficial].*

6.  **Assign Base Color $c_i$:**
    Map the region size ratio $\frac{\sigma_i}{\max_j(\sigma_j)}$ to a color.
    *   *Cold-to-Hot:* Small regions $\rightarrow$ Hot colors (e.g., red); Large regions $\rightarrow$ Cool colors (e.g., blue).
    *   *Pre-defined Array:* Assign from a fixed palette based on region index or size rank.

7.  **Compute Per-Voxel Opacity $\alpha^i_v$:**
    Modulate the region opacity $\alpha_i$ by the local gradient magnitude to enhance boundaries:
    $$ \alpha^i_v = \alpha_i \cdot \frac{\|\nabla v\|}{G_{max, i}} $$

8.  **Compute Per-Voxel Color $c^i_v$:**
    Scale the base color $c_i$ by the local intensity ratio. This scaling is applied only to the **Brightness** component in HSV space to preserve hue/saturation characteristics:
    $$ c^i_v = c_i \cdot \frac{f_v}{F_{max, i}} $$
    *Implementation Note:* Convert $c_i$ to HSV, multiply the V component by the ratio, convert back to RGB.

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $G_{max, i} = 0$ or $F_{max, i} = 0$, handle gracefully (e.g., default to 1.0 ratio or skip modulation) [G].
*   **Identical Sizes:** If $\max_j(\sigma_j) = \min_j(\sigma_j)$, the denominator in Step 4 becomes zero. Add a small epsilon or assign uniform opacity [G].
*   **Occlusion Logic:** The condition $\sigma_i > \sigma_j$ ensures only larger regions occlude smaller ones, preventing circular occlusion dependencies [S1, unofficial].

## Subtopics

### [Automated Visibility-Based TF Parameter Optimization](7072-automated-visibility-based-tf-parameter-optimization.md)
Explicitly enforces quantitative visibility thresholds for Structures of Interest (SOIs) rather than relying on geometric size and occlusion heuristics.

## Operating characteristics
*   **Time Complexity:** Dominated by region growing and statistical computation. For $N$ voxels and $K$ regions, computing means and variances is $O(N)$. Occlusion checks are $O(K^2)$. Pre-processing times reported were ~150-215 seconds for ~256^3 volumes on CPU [S1, unofficial].
*   **Memory Complexity:** Requires storing region membership for all voxels and pre-computed statistics per region. $O(N)$ for voxel data, $O(K)$ for region stats.
*   **Strengths:** Automatically reveals internal structures obscured by larger outer shells without manual tuning. Enhances boundary details via gradient modulation.
*   **Limitations:** Relies on accurate segmentation/clustering. Performance degrades if regions are numerous ($K$ is large) due to $O(K^2)$ occlusion checks. Color assignment is heuristic and may not reflect physical material properties.
*   **When to Use:** Volumetric datasets with distinct, clustered structures where internal visibility is critical (e.g., medical CT/MRI with overlapping tissues).
*   **When Not to Use:** Data without clear spatial clustering or when physical material color is known and required.

## Background
This method was proposed in the context of automatic transfer function design using clustering on LH (Local Histogram) space [S1, unofficial]. It addresses the difficulty of manually tuning transfer functions for complex volumetric data where structures overlap. The approach leverages the geometric properties of segmented clusters to derive visual parameters, ensuring that smaller, potentially occluded structures remain visible.

## Sources
[S1] Automatic Transfer Function Design for Volumetric Data Visualization using Clustering on LH Space, https://www.researchgate.net/publication/265318052_Automatic_Transfer_Function_Design_for_Volumetric_Data_Visualization_using_Clustering_on_LH_Space, fetched 2026-10-28 (unofficial)
