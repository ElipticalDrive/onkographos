# Hierarchical Clustering of Material Boundaries in LH Space

Also called: Automated TF Design using Hierarchical Clustering, Automatic TF Design using Clustering on LH Space, Clustering-based Method for Automatic Transfer Function Design, LH Histogram-based Transfer Function Design, LH Space Clustering, LH histogram clustering, Mean shift clustering for TF design, Transfer Function Design Based on LH Histogram

## Overview
This method automatically generates volume rendering transfer functions (TF) by clustering material boundaries defined in a two-dimensional Lightness-Higher (LH) intensity space histogram [S1][S2, unofficial]. It solves the problem of manual TF specification by identifying boundary clusters, grouping them hierarchically, and assigning visual parameters to minimize occlusion and maximize boundary discrimination [S2, unofficial]. It differs from standard scalar-value/gradient magnitude clustering by using LH space, where boundaries appear as distinct blobs rather than overlapping arches, simplifying cluster shape and reducing ambiguity [S1].

## How it works
The method operates in two main phases: (1) construction of the LH histogram and initial element generation, and (2) hierarchical clustering of these elements to define transfer function regions. A variant approach [S2, unofficial] uses Mean Shift clustering followed by region growing before hierarchical grouping. The following steps describe the core hierarchical clustering framework [S1], with notes on the [S2, unofficial] variant where they diverge.

### Inputs and Outputs
**Inputs:**
- Volumetric dataset $V$ (scalar values).
- Histogram resolution: $256 \times 256$ bins [S1] or $512 \times 512$ pixels [S2, unofficial].
- Parameters: Small cluster threshold (e.g., 10 voxels) [S1], bandwidth $B_w$ (for Mean Shift variant) [S2, unofficial].

**Outputs:**
- A hierarchy of clusters (dendrogram) representing material boundaries.
- Transfer function parameters (color, opacity, gradient multiplier) for each cluster.

### Data Structures
1.  **LH Histogram:** A 2D array of size $N \times N$ (e.g., $256 \times 256$ or $512 \times 512$). Each bin $b_{xy}$ stores the count of voxels with lower intensity $L=x$ and higher intensity $H=y$. In the [S2, unofficial] variant, the histogram is stored as an image where bin values are scaled by the logarithm of weighted sums and mapped to a color band [S2, unofficial].
2.  **Initial Element ($e_i$):** A set of histogram bins belonging to a single local peak.
    -   Size: $|e_i| = \sum_{b_{xy} \in e_i} |b_{xy}|$, where $|b_{xy}|$ is the number of contributions (voxels) in bin $b_{xy}$ [S1].
    -   Representation: Defined by its mean vector $\mu_i = (\mu_{ix}, \mu_{iy})$ and covariance matrix $\Sigma_i$ of a bivariate normal distribution approximating the element [S1].
3.  **Cluster Hierarchy:** A binary tree (dendrogram) where leaves are initial elements and internal nodes represent merged clusters.

### Steps

#### 1. Pre-processing: Gradient and LH Value Computation
Compute the gradient vector and LH values for each voxel.
1.  **Gradient Approximation:** Fit a second-degree polynomial to the density function in a local neighborhood of each voxel. Solve for coefficients by minimizing approximation error. Compute first partial derivatives (gradient $\nabla f$) and second directional derivatives [S2, unofficial]. This accounts for anisotropic voxel spacing without expensive interpolation [S2, unofficial].
2.  **LH Integration:** For each voxel, track boundary paths in both directions (towards lower and higher intensities) using gradient integration.
    -   Use Heun’s method (modified Euler) with step size $d=1$ voxel [S2, unofficial].
    -   Update rule:
        $$ u_{i+1} = u_i + \frac{1}{2}d(\nabla f(u_i) + \nabla f(u_i + d\nabla f(u_i))) $$
        where $u_i$ is the current voxel position, $\nabla f$ is the normalized gradient vector (or its inverse for tracking lower intensity $L$) [S2, unofficial].
    -   Terminate integration when a local extremum or inflection point is reached [S2, unofficial].
    -   Accumulate distances $d_L$ and $d_H$ from the current voxel to the sampled $L$ and $H$ voxels.
3.  **Histogram Accumulation:** For each voxel, determine the bin $(L, H)$. Weight the contribution by:
    $$ w = 1 - \frac{|d_L - d_H|}{d_L + d_H} $$
    Accumulate weighted counts into the LH histogram [S2, unofficial].

#### 2. Initial Element Generation
Identify local maxima (peaks) in the 2D LH histogram. Each peak becomes an initial clustering element $e_i$.
1.  **Blurring:** Blur the histogram with a 2D Gaussian kernel with $\sigma$ equal to the bin size. This removes small noisy local maxima and establishes neighborhood relations between separated peaks [S1].
2.  **Peak Detection:** Label every local maximum. All histogram bins belonging to the same peak form one initial element $e_i$ [S1].
3.  **Small Cluster Removal:** Join any cluster consisting of fewer than 10 voxels with its direct neighbors. This reduces the number of initial elements $n$ from potentially thousands to several hundreds, ensuring interactive performance [S1].

#### 3. Similarity Measure Calculation
Compute the similarity $s(e_i, e_j)$ between every pair of initial elements to build the hierarchy.
1.  **Priors:** Calculate priors based on element sizes:
    $$ P(e_i) = \frac{|e_i|}{|e_i| + |e_j|}, \quad P(e_j) = \frac{|e_j|}{|e_i| + |e_j|} $$
    [S1].
2.  **Bhattacharyya Bound:** Approximate each element $e_i$ as a bivariate normal distribution with mean $\mu_i$ and covariance $\Sigma_i$. Compute the similarity using the Bhattacharyya bound, which estimates the upper bound of the probability of a wrong decision by a Bayesian classifier [S1]:
    $$ s(e_i, e_j) = 2 \sqrt{P(e_i)P(e_j)} e^{-k} $$
    where
    $$ k = \frac{1}{8}(\mu_j - \mu_i)^T \left(\frac{\Sigma_i + \Sigma_j}{2}\right)^{-1} (\mu_j - \mu_i) + \frac{1}{2} \ln \frac{\left|\frac{\Sigma_i + \Sigma_j}{2}\right|}{\sqrt{|\Sigma_i||\Sigma_j|}} $$
    The factor 2 scales the value into the range $[0, 1]$ [S1].
    *Note: A second similarity measure based on spatial connectivity is mentioned but not detailed in the sources [S1].*

#### 4. Hierarchical Clustering
Perform agglomerative hierarchical clustering [S1].
1.  **Initialization:** Each element $e_i$ is in its own cluster $C_i$. The hierarchy has $n$ levels $k=1, \dots, n$. Level $k=1$ has $n$ clusters [S1].
2.  **Iteration:** At each level $k$, find the two most similar clusters $C_i$ and $C_j$:
    $$ s_k = \max \{ s(C_i, C_j) \}; \quad i, j = 1, \dots, n - k + 1; \quad i \neq j $$
    Join these two clusters. Update the similarity matrix for the new merged cluster (e.g., using average linkage or similar, though the specific linkage method is not explicitly defined in the provided text, the maximization step is clear) [S1].
3.  **Termination:** Repeat until one cluster remains at level $k=n$ [S1].

#### 5. Transfer Function Generation (Variant [S2, unofficial])
The [S2, unofficial] variant replaces steps 2-4 with Mean Shift clustering and region growing before generating the TF.
1.  **Mean Shift Clustering:**
    -   Set bandwidth $B_w$ (typically 3%-12% of $\max(L, H)$) [S2, unofficial].
    -   For each point in the LH histogram, find neighbors within $B_w$.
    -   Calculate weighted mean $\mu_n$ of neighbors (weighted by voxel frequency).
    -   Shift window center to $\mu_n$. Repeat until convergence (distance $< \rho B_w$, $\rho=0.001$) [S2, unofficial].
    -   Group points converging to the same mode. Merge clusters with modes within $B_w/2$ [S2, unofficial].
2.  **Bounding Polygon Generation:** Compute convex hulls for each cluster using Andrew’s monotone chain algorithm [S2, unofficial].
3.  **Non-boundary Removal:** Discard clusters where any vertex of the convex hull lies within a diagonal window $H_{diag}$ defined by vertices $[0,0], [0, 0.01 \times \max H], [0.99 \times \max L, \max H], [\max L, \max H], [\max L, 0.99 \times \max H], [0.01 \times \max L, \max H]$ [S2, unofficial].
4.  **Overlap Disambiguation:** For intersecting convex hulls, draw a dividing line between intersections and assign areas to the nearest cluster [S2, unofficial].
5.  **Region Growing:** Apply region growing to incorporate spatial information, filling discontinuities and merging overlapping clusters based on seed points from the clustering step [S2, unofficial].
6.  **TF Assignment:** Assign color and opacity to each cluster, reducing occlusions [S2, unofficial].

### Parameters
-   **Histogram Resolution:** $256 \times 256$ [S1] or $512 \times 512$ [S2, unofficial].
-   **Small Cluster Threshold:** 10 voxels [S1].
-   **Integration Step Size:** 1 voxel [S2, unofficial].
-   **Mean Shift Bandwidth ($B_w$):** 3%-12% of $\max(L, H)$ [S2, unofficial].
-   **Convergence Threshold ($\rho$):** 0.001 [S2, unofficial].
-   **Opacity Parameter ($k_s$):** Used in specific datasets (e.g., Feet) to control skin opacity [S2, unofficial].

### Edge Cases and Pitfalls
-   **Performance:** Too many initial clusters (thousands of peaks) lead to $O(n^2)$ complexity issues. Mitigated by blurring and merging small clusters [S1].
-   **Anisotropy:** The polynomial approximation method accounts for differences between pixel spacing and slice spacing in medical datasets [S2, unofficial].
-   **Noise:** The method is robust to noise because it does not interpolate curves through all data points [S2, unofficial].

## Subtopics

### [Bhattacharyya Similarity for Material Boundaries](6020-bhattacharyya-similarity-for-material-boundaries.md)
A similarity measure between two clustering elements (material boundaries) based on the overlap of their bivariate normal probability density functions (PDFs) in the LH (Lightness-Hue) histogram space. It serves as the basis for hierarchical clustering in the LH space.

### [Transfer Function Generation from Clustering](6023-transfer-function-generation-from-clustering.md)

## Operating characteristics
-   **Time Complexity:** The hierarchical clustering step is $O(n^2)$, where $n$ is the number of initial clustering elements [S1]. Pre-processing (gradient/LH computation) is linear in the number of voxels.
-   **Memory Complexity:** Depends on histogram resolution ($256^2$ or $512^2$ bins) and storage of voxel gradients/LH values [S1][S2, unofficial].
-   **Performance:** Reducing $n$ to several hundreds allows for interactive generation of the hierarchy [S1].
-   **Strengths:** Reduces occlusion, automates TF design, intuitive user interaction with cluster hierarchies, robust to noise and anisotropy [S1][S2, unofficial].
-   **Limitations:** Sensitivity to clustering parameters (mitigated by hierarchical interaction), complexity of overlap resolution in [S2, unofficial] variant [S1][S2, unofficial].
-   **Use Cases:** Medical image visualization, where automatic TF design is critical [S2, unofficial].

## Background
The method was developed by P. Šereda, A. Vilanova, and F. A. Gerritsen at Philips Medical Systems [S1]. It builds on previous research showing that material boundaries form blob-like clusters in LH space [SVSG06, cited in S1]. The hierarchical clustering approach was introduced to facilitate semi-automatic TF design, allowing users to interact with cluster hierarchies instead of tuning thresholds [S1]. A related variant by Binh P. Nguyen et al. [S2, unofficial] refactored mean shift clustering for more stable results in automatic TF design.

## Sources
[S1] Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries, https://doi.org/10.2312/vissym/eurovis06/243-250, fetched 2026-10-26
[S2] Automatic Transfer Function Design for Volumetric Data Visualization using Clustering on LH Space, https://www.researchgate.net/publication/265318052_Automatic_Transfer_Function_Design_for_Volumetric_Data_Visualization_using_Clustering_on_LH_Space, fetched 2026-10-28 (unofficial)
