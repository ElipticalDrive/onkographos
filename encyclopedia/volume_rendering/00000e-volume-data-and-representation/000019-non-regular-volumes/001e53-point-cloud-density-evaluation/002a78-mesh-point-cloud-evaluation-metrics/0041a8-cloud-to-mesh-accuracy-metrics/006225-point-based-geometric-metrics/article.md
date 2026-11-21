# Point-Based Geometric Metrics

Also called: point-to-point, point-to-plane, plane-to-plane

## Overview
Point-Based Geometric Metrics are full-reference methods that assess the geometric fidelity of a distorted point cloud by comparing its points against a reference model [S1]. They differ from projection-based metrics by operating directly in 3D space rather than projecting onto 2D views, and from point-to-mesh metrics by avoiding surface reconstruction artifacts that can skew scores [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Reference point cloud $P_{ref}$, distorted point cloud $P_{dist}$.
*   **Outputs:** A scalar error metric (MSE, RMS, Hausdorff) or PSNR.

**Data Structures:**
*   Points are represented as 3D vectors $\mathbf{p} = (x, y, z)$.
*   Normals are unit vectors $\mathbf{n} = (n_x, n_y, n_z)$.

**Steps:**

1.  **Association:** Associate each point $\mathbf{p}_i \in P_{dist}$ with its nearest neighbor $\mathbf{r}_i \in P_{ref}$ using the nearest neighbor algorithm [S1]. Let $\mathbf{r}_i$ be the reference point associated with $\mathbf{p}_i$.

2.  **Metric Calculation:** Choose one of the following geometric error definitions:

    *   **Point-to-Point:** Computes the Euclidean distance between associated points.
        $$ e_i = \| \mathbf{p}_i - \mathbf{r}_i \|_2 $$
        This reflects geometric displacement from the reference position [S1].

    *   **Point-to-Plane:** Projects the error vector onto the normal vector $\mathbf{n}_i$ of the reference point $\mathbf{r}_i$.
        $$ e_i = | (\mathbf{p}_i - \mathbf{r}_i) \cdot \mathbf{n}_i | $$
        This indicates the deviation of the distorted point from the reference local surface [S1].

    *   **Plane-to-Plane:** Computes the angular similarity between the tangent plane of the distorted point and the tangent plane of the reference point. Let $\mathbf{n}_i^{dist}$ be the normal of the distorted point (estimated from its local neighborhood) and $\mathbf{n}_i$ be the normal of the reference point.
        $$ e_i = 1 - | \mathbf{n}_i^{dist} \cdot \mathbf{n}_i | $$
        This approximates the dissimilarity between corresponding local surfaces [S1].

3.  **Aggregation:** Aggregate the per-point errors $e_i$ for $i=1 \dots N$ using one of the following:
    *   **Mean Squared Error (MSE):**
        $$ \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} e_i^2 $$
    *   **Root-Mean-Squared (RMS):**
        $$ \text{RMS} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} e_i^2} $$
    *   **Hausdorff Distance:**
        $$ H = \max_{i} e_i $$
        (Note: The source mentions Hausdorff distance as a common aggregation method for total geometric degradation [S1].)

4.  **Geometric PSNR:** To account for differently scaled contents, compute the geometric PSNR [S1]:
    $$ \text{PSNR} = 10 \log_{10} \left( \frac{D_{max}^2}{\text{Error}} \right) $$
    Where $D_{max}$ is the squared maximum distance of nearest neighbors of the original (reference) content, and "Error" is the aggregated error value (e.g., MSE) [S1].

**Parameters:**
*   Normal estimation method for Point-to-Plane and Plane-to-Plane: Not given in the sources.
*   Neighborhood size for normal estimation: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Point-to-Mesh:** Metrics based on projected distances to a reconstructed mesh are considered sub-optimal because scores strongly depend on the selected surface reconstruction algorithm [S1]. These are excluded from this family.
*   **Scaling:** Raw error values depend on scene scale; PSNR is required for cross-scene comparison [S1].

## Operating characteristics
*   **Complexity:** Dominated by nearest neighbor search, typically $O(N \log N)$ or $O(N)$ with spatial indexing.
*   **Strengths:** Directly measures 3D geometric displacement or surface deviation.
*   **Limitations:** Requires associated points; sensitive to noise if normals are not robustly estimated.
*   **When to use:** When assessing geometric fidelity of point clouds against a ground truth reference.

## Background
Originating from full-reference quality assessment methodologies for point cloud representations, these metrics are categorized alongside projection-based methods [S1]. The specific distinction between point-to-point, point-to-plane, and plane-to-plane approaches was detailed in studies assessing voxelized point clouds [S1].

## Sources
[S1] A novel methodology for quality assessment of voxelized point clouds, https://doi.org/10.1117/12.2322741, fetched 2026-10-30
