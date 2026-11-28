# Hierarchical Clustering of Material Boundaries in LH Space

Also called: Automated TF Design using Hierarchical Clustering, Automatic TF Design using Clustering on LH Space, Clustering-based Method for Automatic Transfer Function Design, LH Histogram-based Transfer Function Design, LH Space Clustering, LH histogram clustering, Mean shift clustering for TF design, Transfer Function Design Based on LH Histogram

## Overview
This method automates the design of volume rendering transfer functions (TF) by clustering material boundaries defined in the Lower-Higher (LH) intensity space, a 2D histogram domain where boundaries appear as distinct blobs rather than overlapping arches [S1][S2, unofficial]. It solves the problem of manual TF specification by identifying initial clustering elements from histogram peaks and grouping them via hierarchical clustering, allowing users to interact with cluster hierarchies without predefined thresholds [S1]. Unlike standard scalar-value vs. gradient-magnitude spaces, the LH space simplifies cluster shapes and reduces classification ambiguities [S1].

## How it works
The method operates in two primary variants: the original hierarchical clustering approach using Bhattacharyya similarity [S1], and a variant using Mean Shift clustering followed by region growing [S2, unofficial].

### Inputs and Outputs
**Inputs:**
- Volumetric dataset $V$ [S1][S2, unofficial].
- Histogram resolution: $256 \times 256$ bins [S1] or $512 \times 512$ pixels [S2, unofficial].
- Parameters: Small cluster threshold (e.g., 10 voxels) [S1]; Mean shift bandwidth $B_w$ (3%-12% of max LH value) [S2, unofficial]; Convergence threshold $\rho = 0.001$ [S2, unofficial].

**Outputs:**
- A hierarchy of clusters (dendrogram) [S1] or a set of labeled clusters [S2, unofficial].
- A transfer function that maps color and opacity to clusters to reduce occlusions [S2, unofficial].

### Data Structures
1.  **LH Histogram:** A 2D array $H$ of size $N \times N$ (where $N=256$ or $512$) [S1][S2, unofficial]. Each bin $b_{xy}$ stores the count of voxels with lower intensity $L=x$ and higher intensity $H=y$ [S1][S2, unofficial].
2.  **Clustering Element $e_i$:** A set of histogram bins $\{b_{xy}\}$ belonging to a single local peak [S1].
    -   Size: $|e_i| = \sum_{b_{xy} \in e_i} |b_{xy}|$, where $|b_{xy}|$ is the bin count [S1].
    -   Mean vector: $\mu_i = (\mu_{ix}, \mu_{iy}) = \frac{1}{|e_i|} \sum_{b_{xy} \in e_i} (x|b_{xy}|, y|b_{xy}|)$ [S1].
    -   Covariance matrix: $\Sigma_i$ computed from the weighted distribution of bins in $e_i$ [S1].
3.  **Cluster Hierarchy:** A binary tree (dendrogram) representing agglomerative joins [S1].

### Pre-processing
1.  **Gradient Calculation:** Compute gradient vectors $\nabla f$ for each voxel using a second-degree polynomial approximation of the density function in a local neighborhood to determine first partial derivatives [S2, unofficial]. This accounts for anisotropic voxel spacing without expensive interpolation [S2, unofficial].
2.  **LH Value Integration:** Determine lower ($L$) and higher ($H$) intensity values for each voxel by tracking boundary paths using gradient integration in both directions via Heun's method [S2, unofficial].
    -   Use step size $d=1$ voxel [S2, unofficial].
    -   Update rule:
        $$ u_{i+1} = u_i + \frac{1}{2}d(\nabla f(u_i) + \nabla f(u_i + d\nabla f(u_i))) $$
        where $\nabla f$ is the normalized gradient vector (or its inverse for $L$ tracking) [S2, unofficial].
    -   Terminate integration at local extrema or inflection points [S2, unofficial].
3.  **Histogram Construction:**
    -   Accumulate $[L, H]$ pairs into the histogram weighted by $w = 1 - \frac{|d_L - d_H|}{d_L + d_H}$, where $d_L$ and $d_H$ are accumulated distances along the boundary path from the current voxel to the $L$ and $H$ sampling voxels [S2, unofficial].
    -   Scale the sum of weights by taking the logarithm and map to a color band (e.g., cold-to-hot) [S2, unofficial].
4.  **Smoothing:** Blur the histogram with a 2D Gaussian kernel with $\sigma$ equal to the bin size to remove noisy local maxima [S1].

### Method 1: Agglomerative Hierarchical Clustering [S1]
1.  **Initial Element Generation:** Identify local maxima (peaks) in the smoothed 2D LH histogram; each peak becomes an initial clustering element $e_i$ [S1].
2.  **Small Cluster Removal:** Join any cluster consisting of fewer than 10 voxels with its direct neighbors to reduce $n$ to several hundreds for interactive performance [S1].
3.  **Similarity Calculation:** Compute similarity $s(e_i, e_j)$ between all pairs of elements using the Bhattacharyya bound [S1]:
    $$ s(e_i, e_j) = 2 \sqrt{P(e_i)P(e_j)} e^{-k} $$
    where priors are $P(e_i) = \frac{|e_i|}{|e_i| + |e_j|}$ and $P(e_j) = \frac{|e_j|}{|e_i| + |e_j|}$ [S1].
    The exponent $k$ is:
    $$ k = \frac{1}{8}(\mu_j - \mu_i)^T \left(\frac{\Sigma_i + \Sigma_j}{2}\right)^{-1}(\mu_j - \mu_i) + \frac{1}{2} \ln \frac{\left|\frac{\Sigma_i + \Sigma_j}{2}\right|}{\sqrt{|\Sigma_i||\Sigma_j|}} $$
    where $\mu$ and $\Sigma$ are the mean vector and covariance matrix of the bivariate normal approximation of the element [S1].
4.  **Hierarchy Construction:**
    -   Initialize $n$ clusters $C_i = \{e_i\}$ [S1].
    -   For level $k = 1$ to $n-1$:
        -   Find the pair of clusters $(C_i, C_j)$ with maximal similarity $s_k = \max \{s(C_i, C_j)\}$ [S1].
        -   Join $C_i$ and $C_j$ into a new cluster [S1].
    -   Result is a dendrogram [S1].

### Method 2: Mean Shift and Region Growing [S2, unofficial]
1.  **Mean Shift Clustering:**
    -   Set bandwidth $B_w$ (3%-12% of $\max(\max L, \max H)$) [S2, unofficial].
    -   For each point in the LH histogram:
        -   Find neighbors within $B_w$.
        -   Calculate mean $\mu_n$ of neighbors weighted by voxel frequency.
        -   Shift window center to $\mu_n$.
        -   Repeat until convergence (distance between successive means $< \rho B_w$, $\rho=0.001$) [S2, unofficial].
    -   Group points converging to the same mode into a single cluster and merge clusters whose modes are within $B_w/2$ of each other [S2, unofficial].
2.  **Non-Boundary Removal:**
    -   Compute convex hull $H_i$ for each cluster $C_i$ using Andrew’s monotone chain algorithm [S2, unofficial].
    -   Define diagonal window $H_{diag}$ with vertices $[0,0], [0,0.01 \times \max H], [0.99 \times \max L, \max H], [\max L, \max H], [\max L, 0.99 \times \max H], [0.01 \times \max L, \max H]$ [S2, unofficial].
    -   Discard clusters where any vertex of $H_i$ lies within $H_{diag}$ [S2, unofficial].
3.  **Overlap Disambiguation:**
    -   For intersecting convex hulls $H_a$ and $H_b$, compute intersections $I_a$ and $I_b$ [S2, unofficial].
    -   Draw a dividing line between intersections and assign partitioned areas to the nearest cluster [S2, unofficial].
4.  **Region Growing:**
    -   Use clusters from Mean Shift as seed points [S2, unofficial].
    -   Apply region growing to incorporate spatial information, filling discontinuities and merging overlapping clusters based on spatial connectivity [S2, unofficial].

### Pseudocode (Method 1 Core)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot_product(a, b):
    return sum(x * y for x, y in zip(a, b))

def matrix_inverse_2d(M):
    det = M[0][0] * M[1][1] - M[0][1] * M[1][0]
    inv_det = 1.0 / det
    return [[M[1][1] * inv_det, -M[0][1] * inv_det],
            [-M[1][0] * inv_det, M[0][0] * inv_det]]

def matrix_determinant_2d(M):
    return M[0][0] * M[1][1] - M[0][1] * M[1][0]

def compute_bhattacharyya_similarity(elem_i, elem_j):
    p_i = elem_i['size'] / (elem_i['size'] + elem_j['size'])
    p_j = elem_j['size'] / (elem_i['size'] + elem_j['size'])
    mu_i = elem_i['mean']
    mu_j = elem_j['mean']
    sigma_i = elem_i['cov']
    sigma_j = elem_j['cov']
    diff = [mu_j[0] - mu_i[0], mu_j[1] - mu_i[1]]
    avg_sigma = [[(sigma_i[r][c] + sigma_j[r][c]) / 2.0 for c in range(2)] for r in range(2)]
    avg_sigma_inv = matrix_inverse_2d(avg_sigma)
    term1 = 0.125 * dot_product(diff, [sum(avg_sigma_inv[r][c] * diff[c] for c in range(2)) for r in range(2)])
    det_avg = matrix_determinant_2d(avg_sigma)
    det_i = matrix_determinant_2d(sigma_i)
    det_j = matrix_determinant_2d(sigma_j)
    term2 = 0.5 * math.log(det_avg / math.sqrt(det_i * det_j))
    k = term1 + term2
    return 2.0 * math.sqrt(p_i * p_j) * math.exp(-k)

def merge_cluster_stats(cluster):
    total_size = sum(b[2] for b in cluster['bins'])
    mean_x = sum(b[0] * b[2] for b in cluster['bins']) / total_size
    mean_y = sum(b[1] * b[2] for b in cluster['bins']) / total_size
    cov_xx = sum(b[2] * (b[0] - mean_x)**2 for b in cluster['bins']) / total_size
    cov_yy = sum(b[2] * (b[1] - mean_y)**2 for b in cluster['bins']) / total_size
    cov_xy = sum(b[2] * (b[0] - mean_x) * (b[1] - mean_y) for b in cluster['bins']) / total_size
    cluster['size'] = total_size
    cluster['mean'] = [mean_x, mean_y]
    cluster['cov'] = [[cov_xx, cov_xy], [cov_xy, cov_yy]]

def hierarchical_clustering(elements):
    clusters = [{'id': i, 'bins': e['bins'], 'size': e['size'], 'mean': e['mean'], 'cov': e['cov']} for i, e in enumerate(elements)]
    hierarchy = []
    for k in range(1, len(clusters)):
        max_sim = -1.0
        merge_i, merge_j = -1, -1
        for i in range(len(clusters)):
            for j in range(i + 1, len(clusters)):
                sim = compute_bhattacharyya_similarity(clusters[i], clusters[j])
                if sim > max_sim:
                    max_sim = sim
                    merge_i, merge_j = i, j
        new_bins = clusters[merge_i]['bins'] + clusters[merge_j]['bins']
        new_cluster = {'id': k, 'bins': new_bins, 'left': clusters[merge_i], 'right': clusters[merge_j], 'similarity': max_sim}
        merge_cluster_stats(new_cluster)
        hierarchy.append(new_cluster)
        clusters.pop(merge_j)
        clusters.pop(merge_i)
        clusters.append(new_cluster)
    return hierarchy
```

## Subtopics

### [Bhattacharyya Similarity for Material Boundaries](6020-bhattacharyya-similarity-for-material-boundaries.md)
This method computes the similarity between two clustering elements (material boundaries) in the LH space by measuring the overlap of their bivariate normal probability density functions (PDFs). It serves as the primary metric for hierarchical clustering, allowing the system to group visually similar material boundaries without predefined thresholds. Unlike spatial similarity, which relies on voxel connectivity, this method relies solely on the statistical distribution of boundary intensities and hues.

### [Transfer Function Generation from Clustering](6023-transfer-function-generation-from-clustering.md)
This method generates a transfer function (TF) by clustering material boundaries in the LH space and assigning visual properties to the resulting clusters. It solves the problem of manual TF design by automatically grouping similar voxel regions based on hierarchical clustering, allowing users to interact with cluster hierarchies to define color and opacity mappings. It differs from its parent method by explicitly defining the final mapping as a label-based assignment where clusters inherit visual properties upon merging, and from its sibling by focusing on the generation of the TF from the clustering result rather than just measuring similarity.

### LH Histogram Clustering TF
A method building on previous LH histogram work, proposing a clustering-based approach for LH histograms [S3, unofficial]. It is described in reference [26] and utilizes clustering criteria such as distance, separation, and direction of cluster elongation to group clusters together, interpreting them as bivariate 2D PDFs using Bayesian decision theory [S3, unofficial].

## Operating characteristics
-   **Time Complexity:** The complexity of generating the hierarchy is $O(n^2)$, where $n$ is the number of initial clustering elements $e_i$ [S1].
-   **Memory Complexity:** The LH histogram is stored as an image of $512 \times 512$ pixels [S2, unofficial].
-   **Performance:** Reducing the number of initial clusters to several hundreds (via blurring and small cluster merging) allows for interactive generation of the hierarchy [S1].
-   **Strengths:** Reduces occlusions and maximizes boundary discrimination [S2, unofficial]. The LH space facilitates clustering by making boundaries appear as blobs rather than overlapping arches [S1]. Robust to noise because it does not interpolate curves through all data points [S2, unofficial].
-   **Limitations:** Sensitive to the number of initial peaks if not pre-processed; too many peaks lead to poor performance [S1]. Requires tuning of bandwidth $B_w$ in Mean Shift variant [S2, unofficial].
-   **When to Use:** Suitable for medical images and volumetric data where automatic TF design is needed to reduce occlusions [S2, unofficial].
-   **When Not to Use:** When real-time interaction with thousands of fine-grained clusters is required without pre-processing [S1].

## Background
The method was developed by P. Šereda, A. Vilanova, and F. A. Gerritsen at Philips Medical Systems Nederland BV [S1]. It was presented at the Eurographics Symposium on Visualization in 2006 [S1]. The approach builds on previous research showing that material boundaries form blob-like clusters in LH space [S1]. A related variant using Mean Shift clustering was introduced by Binh P. Nguyen et al. in 2012 [S2, unofficial].

## Sources
[S1] Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries, https://doi.org/10.2312/vissym/eurovis06/243-250, fetched 2026-10-26
[S2] Automatic Transfer Function Design for Volumetric Data Visualization using Clustering on LH Space, https://www.researchgate.net/publication/265318052_Automatic_Transfer_Function_Design_for_Volumetric_Data_Visualization_using_Clustering_on_LH_Space, fetched 2026-10-28 (unofficial)
[S3] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
