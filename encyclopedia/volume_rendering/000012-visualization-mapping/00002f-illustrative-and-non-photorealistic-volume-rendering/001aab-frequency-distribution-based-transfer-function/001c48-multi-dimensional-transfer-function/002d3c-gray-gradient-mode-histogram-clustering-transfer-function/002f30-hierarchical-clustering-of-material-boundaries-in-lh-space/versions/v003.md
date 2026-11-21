# Hierarchical Clustering of Material Boundaries in LH Space

Also called: Automated TF Design using Hierarchical Clustering, Automatic TF Design using Clustering on LH Space, Clustering-based Method for Automatic Transfer Function Design, LH Histogram-based Transfer Function Design, LH Space Clustering, LH histogram clustering, Mean shift clustering for TF design, Transfer Function Design Based on LH Histogram

## Overview
This method automates the design of volume rendering transfer functions (TF) by clustering material boundaries defined in the Lightness-Hue (LH) space, a 2D domain where boundaries appear as distinct blobs rather than overlapping arches [S1]. It solves the problem of manual TF specification by identifying initial clustering elements from histogram peaks, grouping them via hierarchical clustering (using Bhattacharyya similarity) or mean-shift clustering, and mapping the resulting clusters to color and opacity [S1][S2, unofficial]. It sets itself apart from standard scalar-gradient clustering by utilizing the LH transformation to simplify cluster shapes and reduce classification ambiguities [S1], and from pure statistical methods by incorporating spatial connectivity and boundary path integration [S2, unofficial].

## How it works
The method operates in two primary variants described in the sources: a hierarchical approach based on peak detection and Bhattacharyya similarity [S1], and a mean-shift approach based on gradient integration and convex hulls [S2, unofficial]. Both variants share the goal of mapping volumetric data to visual properties by clustering voxels based on their scalar values and spatial locations to reduce occlusions [S2, unofficial].

### Inputs and Outputs
*   **Input:** Volumetric dataset (scalar field), optional user interaction parameters (e.g., hierarchy level, opacity scaling factor $k_s$).
*   **Output:** A 2D transfer function mapping LH coordinates to color and opacity, and/or a segmented volume where each voxel is assigned a cluster label.

### Data Structures
1.  **LH Histogram:** Stored as an $N \times N$ pixel image (typically $512 \times 512$) [S2, unofficial]. Each bin $(x, y)$ corresponds to a specific Lower ($L$) and Higher ($H$) intensity pair. The value in the bin is the weighted sum of contributions, scaled by logarithm and mapped to a color band (e.g., cold-to-hot) [S2, unofficial].
2.  **Clustering Elements ($e_i$):** In the hierarchical method, each element is a set of histogram bins belonging to a local peak. The size $|e_i|$ is the sum of contributions in its bins:
    $$ |e_i| = \sum_{b_{xy} \in e_i} |b_{xy}| $$
    where $|b_{xy}|$ is the number of contributions in bin $b_{xy}$ [S1].
3.  **Cluster Bounding Polygons:** In the mean-shift method, each cluster $C_i$ is represented by a convex hull $H_i$ computed using Andrew’s monotone chain algorithm [S2, unofficial].

### Steps

#### 1. Pre-processing: Gradient and LH Value Computation
For each voxel, compute the gradient vector and LH values.
*   **Gradient Approximation:** Use a second-degree polynomial approximation of the density function in a local neighborhood to determine first partial derivatives. This accounts for differences between pixel spacing and slice spacing and avoids expensive interpolation [S2, unofficial].
*   **Gradient Integration (L and H Tracking):** Determine the lower intensity $L$ and higher intensity $H$ for each voxel by tracking boundary paths in both directions using Heun’s method (a modified Euler’s method) [S2, unofficial].
    *   Update rule:
        $$ u_{i+1} = u_i + \frac{1}{2}d(\nabla f(u_i) + \nabla f(u_i + d\nabla f(u_i))) $$
        where $u_i$ is the current voxel position, $\nabla f$ is the normalized gradient vector (or its inverse for L tracking), and $d$ is the step size (typically 1 voxel) [S2, unofficial].
    *   Termination: Stop integration when a local extremum or an inflexion point is reached [S2, unofficial].
*   **Weighting:** Accumulate $[L, H]$ pairs into the histogram with a weight factor $w$:
    $$ w = 1 - \frac{|d_L - d_H|}{d_L + d_H} $$
    where $d_L$ and $d_H$ are the accumulated distances along the boundary path from the current voxel to the sampling voxels corresponding to $L$ and $H$ [S2, unofficial].

#### 2. Initial Element Generation
*   **Histogram Blurring:** Blur the LH histogram with a 2D Gaussian kernel with $\sigma$ equal to the bin size. This removes small noisy local maxima and establishes neighborhood relations [S1].
*   **Peak Detection:** Identify local maxima (peaks) in the 2D LH histogram. Each peak becomes an initial clustering element $e_i$. All histogram bins belonging to the same peak form one element [S1].
*   **Small Cluster Merging (Hierarchical Variant):** Join any cluster consisting of fewer than 10 voxels with its direct neighbors to reduce the number of initial elements $n$ to several hundreds for interactive performance [S1].

#### 3. Clustering
Two distinct clustering strategies are described:

**Variant A: Hierarchical Clustering with Bhattacharyya Similarity [S1]**
1.  **Initialization:** Each element $e_i$ starts in its own cluster $C_i$. The hierarchy has $n$ levels $k = 1, \dots, n$. Level $k=1$ has $n$ clusters.
2.  **Similarity Calculation:** Compute the similarity $s(e_i, e_j)$ between every pair of elements using the Bhattacharyya bound, treating elements as bivariate normal distributions with mean $\mu$ and covariance $\Sigma$:
    $$ s(e_i, e_j) = 2 \sqrt{P(e_i)P(e_j)} e^{-k} $$
    where priors are $P(e_i) = \frac{|e_i|}{|e_i| + |e_j|}$ and $k$ is the Mahalanobis-like distance term:
    $$ k = \frac{1}{8}(\mu_j - \mu_i)^T \left(\frac{\Sigma_i + \Sigma_j}{2}\right)^{-1} (\mu_j - \mu_i) + \frac{1}{2} \ln \frac{\left|\frac{\Sigma_i + \Sigma_j}{2}\right|}{\sqrt{|\Sigma_i||\Sigma_j|}} $$
3.  **Agglomeration:** At each level $k$, find the maximal similarity $s_k$ between any two distinct clusters $C_i, C_j$:
    $$ s_k = \max \{ s(C_i, C_j) \}; \quad i, j = 1, \dots, n - k + 1; \quad i \neq j $$
    Join the two most similar clusters. Repeat until one cluster remains.

**Variant B: Mean-Shift Clustering [S2, unofficial]**
1.  **Setup:** Set window bandwidth $B_w$ (typically 3%-12% of $maxLH = \max(\max L, \max H)$).
2.  **Iteration:** For each point in the LH histogram, find neighbors within $B_w$. Calculate the weighted mean $\mu_n$ of neighbors (weighted by voxel frequency). Shift the window center to $\mu_n$. Repeat until convergence (distance between successive means $< \rho B_w$, where $\rho = 0.001$) [S2, unofficial].
3.  **Grouping:** Group points converging to the same mode into a single cluster. Merge clusters whose modes are within $B_w/2$ of each other [S2, unofficial].

#### 4. Post-processing and TF Generation
*   **Bounding Polygon Generation (Mean-Shift Variant):** Compute the convex hull $H_i$ for each cluster $C_i$ [S2, unofficial].
*   **Non-Boundary Removal:** Define a diagonal window $H_{diag}$ with vertices $[0,0], [0, 0.01 \times maxH], [0.99 \times maxL, maxH], [maxL, maxH], [maxL, 0.99 \times maxH], [0.01 \times maxL, maxH]$. Discard clusters where any vertex of $H_i$ lies within $H_{diag}$ (width 2% of LH range) [S2, unofficial].
*   **Overlap Disambiguation:** For intersecting convex hulls $H_a$ and $H_b$, compute intersections $I_a$ and $I_b$. Draw a dividing line between intersections and assign partitioned areas to the nearest cluster [S2, unofficial].
*   **Region Growing (Mean-Shift Variant):** Apply region growing to incorporate spatial information, filling discontinuities and merging overlapping clusters based on seed points from mean shift clusters [S2, unofficial].
*   **TF Assignment:** Assign color and opacity to clusters. Opacity can be controlled by a parameter $k_s$ (e.g., decreasing skin opacity in the Feet dataset) [S2, unofficial].

### Parameters
*   **Histogram Resolution:** 256x256 bins (Hierarchical) [S1] or 512x512 pixels (Mean-Shift) [S2, unofficial].
*   **Small Cluster Threshold:** < 10 voxels (Hierarchical) [S1].
*   **Integration Step Size $d$:** 1 voxel [S2, unofficial].
*   **Mean-Shift Bandwidth $B_w$:** 3%-12% of $maxLH$ [S2, unofficial].
*   **Convergence Threshold $\rho$:** 0.001 [S2, unofficial].
*   **Diagonal Window Width:** 2% of LH range [S2, unofficial].

### Edge Cases and Pitfalls
*   **Performance:** Too many initial clusters (thousands of peaks) lead to $O(n^2)$ complexity issues. Mitigated by blurring and merging small clusters [S1].
*   **Anisotropy:** The polynomial approximation method accounts for differences between pixel spacing and slice spacing in medical datasets [S2, unofficial].
*   **Noise:** The method is robust to noise because it does not interpolate the curve passing through all data points [S2, unofficial].

## Subtopics

### [Bhattacharyya Similarity for Material Boundaries](6020-bhattacharyya-similarity-for-material-boundaries.md)
Computes similarity between clustering elements by measuring the overlap of their bivariate normal PDFs in LH space, serving as the primary metric for hierarchical clustering without predefined thresholds.

### [Transfer Function Generation from Clustering](6023-transfer-function-generation-from-clustering.md)
Generates a 2D transfer function by assigning color and opacity values to clusters identified in LH space, leveraging the hierarchical structure to allow user selection of hierarchy levels for material boundary definition.

## Operating characteristics
*   **Time Complexity:** The hierarchical clustering step has complexity $O(n^2)$, where $n$ is the number of initial clustering elements [S1]. Mean-shift clustering complexity depends on the number of histogram points and iterations but is optimized by computing over discrete LH values weighted by voxel frequency [S2, unofficial].
*   **Memory Complexity:** Dominated by the storage of the LH histogram ($N \times N$ image) and the volume data. The histogram is stored as a $512 \times 512$ pixel image [S2, unofficial].
*   **Performance:** Reducing initial clusters to several hundreds allows for interactive generation of the hierarchy [S1].
*   **Strengths:** Reduces occlusions, automates TF specification, intuitive user interaction with cluster hierarchies, robust to noise, handles anisotropic data [S1][S2, unofficial].
*   **Limitations:** Sensitivity to clustering parameters (e.g., $B_w$) in mean-shift variant; requires post-processing (region growing) for spatial coherence in mean-shift variant [S2, unofficial].
*   **When to Use:** Medical image visualization where automatic TF design is needed to reduce occlusions and improve efficiency [S2, unofficial].
*   **When Not to Use:** When real-time performance is critical and $n$ is very large without pre-filtering; when spatial coherence is not required and simple scalar clustering suffices.

## Background
The method originates from research by Šereda, Vilanova, and Gerritsen (2006) on automating TF design using hierarchical clustering of material boundaries [S1]. It builds on the concept of LH space, where boundaries appear as blobs, reducing overlap compared to scalar-gradient space [S1]. A later variant by Nguyen et al. (2012) introduced mean-shift clustering and region growing for automatic TF design [S2, unofficial]. The LH space approach was previously explored by Lundström et al. (2006) [S1][S2, unofficial].

## Sources
[S1] Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries, https://doi.org/10.2312/vissym/eurovis06/243-250, fetched 2026-10-26
[S2] Automatic Transfer Function Design for Volumetric Data Visualization using Clustering on LH Space, https://www.researchgate.net/publication/265318052_Automatic_Transfer_Function_Design_for_Volumetric_Data_Visualization_using_Clustering_on_LH_Space, fetched 2026-10-28 (unofficial)
