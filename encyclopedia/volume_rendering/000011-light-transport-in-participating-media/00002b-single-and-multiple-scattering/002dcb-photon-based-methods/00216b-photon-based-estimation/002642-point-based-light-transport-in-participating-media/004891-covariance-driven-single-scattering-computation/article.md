# Covariance Driven Single Scattering Computation

Also called: Frequency Analysis for Single Scattering, Kernel Prediction with Covariance

## Overview
This method extends point-based light transport for participating media with refractive boundaries by replacing fixed kernel sizes with dynamic kernels predicted by the local light field's frequency content [S1]. It solves the problem of over-blurred single scattering in high-frequency volume caustics by using the determinant of a covariance matrix to adaptively shrink the sampling kernel where light transport is complex, allowing fewer samples to capture sharp details [S1]. It sets itself apart from its parent method [Point-Based Light Transport in Participating Media](3763-point-based-light-transport-in-participating-media.md) by specifically optimizing the single scattering pass with frequency analysis, while retaining the parent's approach for double and multiple scattering [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A homogeneous participating media enclosed in refractive boundaries; surface samples distributed on the media boundary; extinction coefficient $\sigma_t$; parameters $C_k$ and $\alpha$.
*   **Outputs:** Single scattering radiance for camera rays.

**Data Structures:**
*   **Volume Sample:** Stores radiance, position, and the determinant of the covariance matrix ($\eta$).
*   **Spatial Hierarchy:** An octree or tree structure organizing volume samples. Each node contains an Axis-Aligned Bounding Box (AABB). Leaf nodes contain the actual volume samples and their Oriented Bounding Boxes (OBB).

**Precomputation:**
1.  Distribute surface samples on the media boundary.
2.  Refract light rays from surface samples into the volume.
3.  Sample refracted rays to create volume samples.
4.  For each volume sample, compute the covariance matrix $\Sigma$ of the local light field.
    *   Initialize $\Sigma$ at the light source.
    *   Update $\Sigma$ using transport operators: free space travel, projection, boundary refraction, and medium travel [S1].
5.  Compute the frequency factor $\eta$ as the determinant of $\Sigma$:
    $$ \eta = \sqrt{|\Sigma|} $$
    The value $\eta$ ranges from 0 to 1, where higher values indicate larger frequency content (sharper caustics) [S1].
6.  Store $\eta$ and radiance for each volume sample.
7.  Organize samples into a spatial hierarchy (octree/tree) with AABBs for internal nodes and OBBs for leaf nodes [S1].

**Rendering:**
1.  Refract the camera ray into the volume.
2.  Sample the refracted camera ray at regular intervals to create camera samples.
3.  For each camera sample, traverse the volume sample hierarchy from the root:
    *   Check if the node's AABB contains the camera sample.
    *   If not, discard the node.
    *   If yes, recursively descend until leaf nodes are reached.
4.  For each leaf node whose AABB contains the camera sample, perform a kernel test using the leaf's OBB.
5.  If the OBB contains the camera sample, the volume sample contributes to the radiance.
6.  Calculate the kernel multiplier $k_i$ to adjust the kernel size based on frequency:
    $$ k_i = C_k (\alpha \cos(\eta \pi) + (1 - \alpha)) $$
    where $0 < \alpha < 1$ [S1].
7.  Update the radiance of the volume sample by multiplying it with $k_i$ to conserve energy [S1].
8.  Evaluate single scattering radiance using the adjusted kernel [S1].

**Parameters:**
*   $C_k$: Constant factor controlling kernel size. Value: $20 \times \text{clamp}(\sigma_t, 0.3, 0.8)$, where $\sigma_t$ is the extinction coefficient [S1].
*   $\alpha$: Tweakable parameter. Typical value: 0.4 [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The determinant calculation for $\eta$ must be stable. $\eta$ is clamped or normalized to the range [0, 1] [S1].
*   **Boundaries:** The method assumes homogeneous media and refractive boundaries. Non-homogeneous media or non-refractive boundaries require different transport operators for $\Sigma$ [S1].
*   **Degenerate Input:** If $\eta = 0$ (uniform distribution), the kernel size is maximized ($k_i \approx C_k$). If $\eta = 1$ (high frequency), the kernel size is minimized ($k_i \approx C_k(1-\alpha)$) [S1].

## Subtopics

### Covariance Analysis of Light Transport
A framework for frequency analysis of light transport that represents the Fourier spectrum of the local light field using a covariance matrix to predict frequency content and adapt sampling/reconstruction [S1]. The local light field is defined as a 4D function parameterized by two spatial coordinates ($\delta x, \delta y$) and two angular coordinates ($\delta \theta, \delta \phi$) around a central ray [S1]. It computes eigenvectors and eigenvalues of $\Sigma$ to determine principal spreading directions and variance [S1]. Transport operators (free space, reflection, refraction) are applied as matrix operations on $\Sigma$ to propagate frequency information [S1]. The covariance matrix $\Sigma$ is defined as:
$$ \Sigma_{i,j} = \frac{\int_{\Omega} (x \cdot e_i)(x \cdot e_j) f(x) dx}{\int_{\Omega} f(x) dx}, \quad \forall (i, j) \in \{1, \dots, 4\}^2 $$
where $e_i$ is the $i$-th vector of the canonical basis of the 4D space $\Omega$ and $x \cdot y$ is the dot product [S1]. This framework is used to compute the covariance matrix for volume samples in the main method [S1].

## Operating characteristics
*   **Time Complexity:** Approximately twice as fast as Wang et al. [3] for equal quality requirements [S1].
*   **Memory Complexity:** Uses one-fifth of the memory cost of Wang et al. [3] for equal quality [S1].
*   **Measured Performance:** Tested on a 2.20GHz Intel Xeon (16 cores), 24 GB RAM, at 512x512 resolution [S1].
*   **Strengths:** Produces sharper results (visually identical to UPBP reference) with equal computation time compared to Wang et al. [3], which produces over-blurred results [S1].
*   **Limitations:** Performance may vary with scene complexity; however, it is less dependent on scene complexity than Holzschuch [2] [S1].
*   **When to Use:** When rendering homogeneous participating media with refractive boundaries and high-frequency volume caustics, especially when memory or computation time is constrained compared to the original point-based method.
*   **When Not to Use:** For non-homogeneous media or media without refractive boundaries, as the transport operators for $\Sigma$ would need significant modification [S1].

## Background
The method was introduced by Durand et al. [4] and Belcour et al. [5, 6] as a framework for frequency analysis of light transport [S1]. It extends the work of Wang et al. [3] on point-based light transport in participating media with refractive boundaries [S1]. The authors implemented the algorithm in the Mitsuba Renderer [14] and compared it against Wang et al. [3], Holzschuch [2], and Unified Points, Beams and Paths (UPBP) by Křivánek et al. [16] [S1].

## Sources
[S1] Fast Computation of Single Scattering in Participating Media with Refractive Boundaries Using Frequency Analysis, https://doi.org/10.1109/tvcg.2019.2909875, fetched 2026-10-26
