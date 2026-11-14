# Bhattacharyya Similarity for Material Boundaries

Also called: LH similarity, PDF overlap similarity

## Overview
This method computes the similarity between two clustering elements (material boundaries) in a volume rendering transfer function design framework by measuring the overlap of their bivariate normal probability density functions (PDFs) in the Lightness-Hue (LH) histogram space [S1]. It serves as the primary metric for hierarchical clustering, grouping visually similar material boundaries based on statistical distribution overlap rather than spatial connectivity [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of initial clustering elements $e_i$, where each element is a set of histogram bins $b_{xy}$ in the LH space [S1]. Each bin $b_{xy}$ has a count $|b_{xy}|$ representing the number of contributions (voxels) at discrete position $(x, y)$ [S1].
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
    Compute the variances and covariance [S1]. To prevent singular matrices for elements that are only 1 bin wide, add a regularization term of $1/16$ to the diagonal elements [S1].
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
import math

def calculate_element_statistics(bins):
    """
    bins: list of tuples (x, y, count)
    Returns: size, mean_vector, covariance_matrix
    """
    if not bins:
        return 0, [0.0, 0.0], [[1.0, 0.0], [0.0, 1.0]]  # [G] Guard for empty input

    xs = [b[0] for b in bins]
    ys = [b[1] for b in bins]
    counts = [b[2] for b in bins]

    size = sum(counts)
    if size == 0:
        return 0, [0.0, 0.0], [[1.0, 0.0], [0.0, 1.0]]

    mu_x = sum(x * c for x, c in zip(xs, counts)) / size
    mu_y = sum(y * c for y, c in zip(ys, counts)) / size
    mu = [mu_x, mu_y]

    dx = [x - mu_x for x in xs]
    dy = [y - mu_y for y in ys]

    sigma_xx = sum(d**2 * c for d, c in zip(dx, counts)) / size + 1/16
    sigma_yy = sum(d**2 * c for d, c in zip(dy, counts)) / size + 1/16
    sigma_xy = sum(d1 * d2 * c for d1, d2, c in zip(dx, dy, counts))

    sigma = [[sigma_xx, sigma_xy], [sigma_xy, sigma_yy]]
    return size, mu, sigma

def det2x2(M):
    return M[0][0] * M[1][1] - M[0][1] * M[1][0]

def inv2x2(M):
    d = det2x2(M)
    if d == 0:
        return [[1.0, 0.0], [0.0, 1.0]]  # [G] Fallback for numerical singularity
    inv_d = 1.0 / d
    return [[M[1][1] * inv_d, -M[0][1] * inv_d],
            [-M[1][0] * inv_d, M[0][0] * inv_d]]

def bhattacharyya_similarity(element1_bins, element2_bins):
    size1, mu1, sigma1 = calculate_element_statistics(element1_bins)
    size2, mu2, sigma2 = calculate_element_statistics(element2_bins)

    total_size = size1 + size2
    if total_size == 0:
        return 0.0

    p1 = size1 / total_size
    p2 = size2 / total_size

    diff = [mu2[0] - mu1[0], mu2[1] - mu1[1]]

    avg_sigma = [[(sigma1[i][j] + sigma2[i][j]) / 2.0 for j in range(2)] for i in range(2)]

    inv_avg = inv2x2(avg_sigma)
    quad_term = 0.125 * (diff[0] * (inv_avg[0][0] * diff[0] + inv_avg[0][1] * diff[1]) +
                         diff[1] * (inv_avg[1][0] * diff[0] + inv_avg[1][1] * diff[1]))

    det_avg = det2x2(avg_sigma)
    det1 = det2x2(sigma1)
    det2_val = det2x2(sigma2)

    if det_avg <= 0 or det1 <= 0 or det2_val <= 0:
        log_term = 0.0
    else:
        log_term = 0.5 * math.log(det_avg / math.sqrt(det1 * det2_val))

    k = quad_term + log_term
    similarity = 2.0 * math.sqrt(p1 * p2) * math.exp(-k)
    return similarity
```

**Edge Cases and Pitfalls**
*   **Singular Covariance:** If an element consists of only a single bin, the covariance matrix would be singular [S1]. This is handled by the $+1/16$ regularization term on the diagonal variances, which is mathematically equivalent to splitting each bin into 4 sub-bins at half-pixel offsets [S1].
*   **Range Scaling:** The factor of 2 in the similarity formula is explicitly required to scale the result into the interval $[0, 1]$ [S1].
*   **Semi-metric Property:** The similarity measure is positive, symmetric, and reflexive, but does not satisfy the triangle inequality [S1]. This is acceptable because the algorithm only compares similarity values for clustering decisions, not performing arithmetic operations on them [S1].

## Subtopics

### Spatial Similarity in the Volume
A similarity measure designed to group boundaries that are connected in the volume, serving as an alternative to LH space similarity [S1]. It evaluates the number of direct neighborhood relations between clusters [S1]. The unweighted neighborhood count is $NR(e_i, e_j) = \sum_{v_i \in e_i} \sum_{v_j \in e_j} N_{26}(v_i, v_j)$, where $N_{26}$ is 1 if voxels are 26-neighbors and 0 otherwise [S1]. The weighted relation is $R(e_i, e_j) = \sum_{v_i \in e_i} \sum_{v_j \in e_j} N_{26}(v_i, v_j) r(v_i, v_j)$, where $r(v_i, v_j) \in [0, 1]$ reflects directional coherence [S1]. When clusters $C_i$ and $C_j$ join, relations update as $NR(C_{i \cup j}) = NR(C_i) + NR(C_j) - 2NR(C_i, C_j)$, $NR(C_{i \cup j}, C_k) = NR(C_i, C_k) + NR(C_j, C_k)$, and $R(C_{i \cup j}, C_k) = R(C_i, C_k) + R(C_j, C_k)$ [S1]. The similarity is normalized by total relations to be size-independent: $s(e_i, e_j) = \max\left(\frac{R(e_i, e_j)}{NR(e_i)}, \frac{R(e_j, e_i)}{NR(e_j)}\right)$ [S1]. Use this variant when spatial connectivity is more important than histogram proximity, or to combine with LH similarity to resolve ambiguities where boundaries are close in LH space but distinct in volume [S1].

## Operating characteristics
*   **Complexity:** The complexity of joining two clusters is $O(K)$, where $K$ is the number of remaining clusters in the hierarchy [S1]. Calculating the similarity for a pair of elements involves matrix inversion and determinant calculation, which is constant time for bivariate distributions ($2 \times 2$ matrices) [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** It provides a probabilistic basis for clustering that accounts for the spread and overlap of material distributions, not just their centroids [S1]. The regularization ensures numerical stability for small clusters [S1].
*   **Limitations:** It ignores spatial context within the volume, potentially grouping boundaries that are statistically similar but spatially disconnected [S1]. It does not satisfy the triangle inequality, limiting its use in metric-based search structures [S1].
*   **When to use:** When designing transfer functions based on the statistical properties of material boundaries in the LH histogram space [S1].

## Background
The method was introduced by P. Šereda, A. Vilanova, and F. A. Gerritsen in 2006 as part of a framework for automating transfer function design using hierarchical clustering of material boundaries [S1]. It is based on the Bhattacharyya bound, which serves as an upper estimate of the probability of a wrong decision made by a Bayesian classifier given the PDFs and priors [S1].

## Sources
[S1] Automating Transfer Function Design for Volume Rendering Using Hierarchical Clustering of Material Boundaries, https://doi.org/10.2312/vissym/eurovis06/243-250, fetched 2026-10-26
