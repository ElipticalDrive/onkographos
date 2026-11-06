# Bhattacharyya Similarity for Material Boundaries

Also called: LH similarity, PDF overlap similarity

## Overview
This method computes the similarity between two clustering elements (material boundaries) in a volume rendering transfer function design framework by measuring the overlap of their bivariate normal probability density functions (PDFs) in the Lightness-Hue (LH) histogram space [S1]. It serves as the primary metric for hierarchical clustering, allowing the system to group visually similar material boundaries without predefined thresholds [S1]. Unlike spatial similarity, which relies on voxel connectivity, this method relies solely on the statistical distribution of boundary intensities and hues [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of initial clustering elements $e_i$, where each element is a set of histogram bins $b_{xy}$ in the LH space. Each bin $b_{xy}$ has a count $|b_{xy}|$ representing the number of contributions (voxels) at discrete position $(x, y)$ [S1].
*   **Output:** A similarity value $s(e_i, e_j) \in [0, 1]$ between any two elements [S1].

**Data Structures**
Each element $e_i$ is characterized by:
1.  **Size:** $|e_i| = \sum_{b_{xy} \in e_i} |b_{xy}|$ [S1].
2.  **Mean Vector:** $\mu_i = [\mu_{ix}, \mu_{iy}]^T$, representing the centroid of the distribution in LH space [S1].
3.  **Covariance Matrix:** $\Sigma_i$, a $2 \times 2$ symmetric matrix describing the spread of the distribution [S1].
4.  **Prior Probability:** $P(e_i)$, derived from the relative size of the element compared to another element $e_j$ [S1].

**Numbered Steps**

1.  **Calculate Element Size:**
    For each element $e_i$, sum the contributions of all bins belonging to it:
    $$ |e_i| = \sum_{b_{xy} \in e_i} |b_{xy}| $$
    [S1]

2.  **Calculate Mean Vector:**
    Compute the weighted average position of the bins in LH space:
    $$ \mu_{ix} = \frac{1}{|e_i|} \sum_{b_{xy} \in e_i} x |b_{xy}| $$
    $$ \mu_{iy} = \frac{1}{|e_i|} \sum_{b_{xy} \in e_i} y |b_{xy}| $$
    $$ \mu_i = [\mu_{ix}, \mu_{iy}]^T $$
    [S1]

3.  **Calculate Covariance Matrix:**
    Compute the variances and covariance. To prevent singular matrices for elements that are only 1 bin wide, add a regularization term of $1/16$ to the diagonal elements [S1].
    $$ \sigma_{i,xx} = \frac{1}{|e_i|} \sum_{b_{xy} \in e_i} (x - \mu_{ix})^2 |b_{xy}| + \frac{1}{16} $$
    $$ \sigma_{i,yy} = \frac{1}{|e_i|} \sum_{b_{xy} \in e_i} (y - \mu_{iy})^2 |b_{xy}| + \frac{1}{16} $$
    $$ \sigma_{i,xy} = \sigma_{i,yx} = \frac{1}{|e_i|} \sum_{b_{xy} \in e_i} (x - \mu_{ix})(y - \mu_{iy}) |b_{xy}| $$
    $$ \Sigma_i = \begin{pmatrix} \sigma_{i,xx} & \sigma_{i,xy} \\ \sigma_{i,yx} & \sigma_{i,yy} \end{pmatrix} $$
    [S1]

4.  **Calculate Priors:**
    For a pair of elements $e_i$ and $e_j$, calculate their priors based on their relative sizes:
    $$ P(e_i) = \frac{|e_i|}{|e_i| + |e_j|} $$
    $$ P(e_j) = \frac{|e_j|}{|e_i| + |e_j|} $$
    [S1]

5.  **Calculate Bhattacharyya Distance ($k$):**
    Compute the distance between the two bivariate normal distributions defined by $(\mu_i, \Sigma_i)$ and $(\mu_j, \Sigma_j)$:
    $$ k = \frac{1}{8} (\mu_j - \mu_i)^T \left(\frac{\Sigma_i + \Sigma_j}{2}\right)^{-1} (\mu_j - \mu_i) + \frac{1}{2} \ln \frac{\left|\frac{\Sigma_i + \Sigma_j}{2}\right|}{\sqrt{|\Sigma_i||\Sigma_j|}} $$
    where $|\cdot|$ denotes the determinant of the matrix [S1].

6.  **Compute Similarity ($s$):**
    Apply the Bhattacharyya bound formula, scaled by 2 to ensure the output range is $[0, 1]$:
    $$ s(e_i, e_j) = 2 \sqrt{P(e_i)P(e_j)} e^{-k} $$
    [S1]

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def calculate_element_statistics(bins):
    """
    bins: list of tuples (x, y, count)
    Returns: size, mean_vector, covariance_matrix
    """
    if not bins:
        return 0, np.zeros(2), np.eye(2)
    
    xs = np.array([b[0] for b in bins])
    ys = np.array([b[1] for b in bins])
    counts = np.array([b[2] for b in bins])
    
    size = np.sum(counts)
    
    # Mean
    mu_x = np.sum(xs * counts) / size
    mu_y = np.sum(ys * counts) / size
    mu = np.array([mu_x, mu_y])
    
    # Covariance with regularization
    dx = xs - mu_x
    dy = ys - mu_y
    
    sigma_xx = np.sum(dx**2 * counts) / size + 1/16
    sigma_yy = np.sum(dy**2 * counts) / size + 1/16
    sigma_xy = np.sum(dx * dy * counts) / size
    
    sigma = np.array([[sigma_xx, sigma_xy],
                      [sigma_xy, sigma_yy]])
    
    return size, mu, sigma

def bhattacharyya_similarity(element1_bins, element2_bins):
    """
    Computes similarity between two elements defined by their histogram bins.
    """
    size1, mu1, sigma1 = calculate_element_statistics(element1_bins)
    size2, mu2, sigma2 = calculate_element_statistics(element2_bins)
    
    # Priors
    total_size = size1 + size2
    if total_size == 0:
        return 0.0
    p1 = size1 / total_size
    p2 = size2 / total_size
    
    # Bhattacharyya Distance k
    diff = mu2 - mu1
    
    # Mean covariance matrix
    avg_sigma = (sigma1 + sigma2) / 2
    
    # Quadratic term
    try:
        inv_avg_sigma = np.linalg.inv(avg_sigma)
        quad_term = 0.125 * diff @ inv_avg_sigma @ diff
    except np.linalg.LinAlgError:
        # Fallback if singular despite regularization (should not happen with +1/16)
        quad_term = 0.0
        
    # Log determinant term
    det_avg = np.linalg.det(avg_sigma)
    det1 = np.linalg.det(sigma1)
    det2 = np.linalg.det(sigma2)
    
    if det_avg <= 0 or det1 <= 0 or det2 <= 0:
        log_term = 0.0
    else:
        log_term = 0.5 * np.log(det_avg / np.sqrt(det1 * det2))
        
    k = quad_term + log_term
    
    # Similarity
    similarity = 2 * np.sqrt(p1 * p2) * np.exp(-k)
    
    return similarity
```

**Edge Cases and Pitfalls**
*   **Singular Covariance:** If an element consists of only a single bin, the covariance matrix would be singular. This is handled by the $+1/16$ regularization term on the diagonal variances, which is mathematically equivalent to splitting each bin into 4 sub-bins at half-pixel offsets [S1].
*   **Range Scaling:** The factor of 2 in the similarity formula is explicitly required to scale the result into the interval $[0, 1]$ [S1].
*   **Semi-metric Property:** The similarity measure is positive, symmetric, and reflexive, but does not satisfy the triangle inequality. This is acceptable because the algorithm only compares similarity values for clustering decisions, not performing arithmetic operations on them [S1].

## Subtopics

### Spatial Similarity in the Volume
A similarity measure designed to group boundaries that are connected in the volume, serving as an alternative to LH space similarity [S1]. It evaluates the number of direct neighborhood relations between clusters [S1].
*   **Definition:** It counts 26-neighbor voxel relations between elements, optionally weighted by directional coherence $r(v_i, v_j)$ [S1].
*   **Formula:** The unweighted neighborhood count is $NR(e_i, e_j) = \sum_{v_i \in e_i} \sum_{v_j \in e_j} N_{26}(v_i, v_j)$, where $N_{26}$ is 1 if voxels are neighbors [S1]. The weighted relation is $R(e_i, e_j) = \sum_{v_i \in e_i} \sum_{v_j \in e_j} N_{26}(v_i, v_j) r(v_i, v_j)$ [S1].
*   **Cluster Update:** When clusters $C_i$ and $C_j$ join, the relations are updated as: $NR(C_{i \cup j}) = NR(C_i) + NR(C_j) - 2NR(C_i, C_j)$ and $NR(C_{i \cup j}, C_k) = NR(C_i, C_k) + NR(C_j, C_k)$ [S1].
*   **Similarity Calculation:** The similarity is normalized by the total relations to be size-independent: $s(e_i, e_j) = \max\left(\frac{R(e_i, e_j)}{NR(e_i)}, \frac{R(e_j, e_i)}{NR(e_j)}\right)$ [S1].
*   **When to choose:** Use when spatial connectivity is more important than histogram proximity, or to combine with LH similarity to resolve ambiguities where boundaries are close in LH space but distinct in volume [S1].

## Operating characteristics
*   **Complexity:** The complexity of joining two clusters is $O(K)$, where $K$ is the number of remaining clusters in the hierarchy [S1]. Calculating the similarity for a pair of elements involves matrix inversion and determinant calculation, which is constant time for bivariate distributions ($2 \times 2$ matrices) [S1].
*   **Strengths:** It provides a probabilistic basis for clustering that accounts for the spread and overlap of material distributions, not just their centroids [S1]. The regularization ensures numerical stability for small clusters [S1].
*   **Limitations:** It ignores spatial context within the volume, potentially grouping boundaries that are statistically similar but spatially disconnected [S1]. It does not satisfy the triangle inequality, limiting its use in metric-based search structures [S1].
*   **When to use:** When designing transfer functions based on the statistical properties of material boundaries in the LH histogram space [S1].

## Background
The method was introduced by P. Šereda, A. Vilanova, and F. A. Gerritsen in 2006 as part of a framework for automating transfer function design using hierarchical clustering of material boundaries [S1]. It is based on the Bhattacharyya bound, which serves as an upper estimate of the probability of a wrong decision made by a Bayesian classifier given the PDFs and priors [S1]. The approach allows for intuitive user interaction with cluster hierarchies without requiring predefined thresholds [S1].

## Sources
[S1] Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries, https://doi.org/10.2312/vissym/eurovis06/243-250, fetched 2026-10-26
