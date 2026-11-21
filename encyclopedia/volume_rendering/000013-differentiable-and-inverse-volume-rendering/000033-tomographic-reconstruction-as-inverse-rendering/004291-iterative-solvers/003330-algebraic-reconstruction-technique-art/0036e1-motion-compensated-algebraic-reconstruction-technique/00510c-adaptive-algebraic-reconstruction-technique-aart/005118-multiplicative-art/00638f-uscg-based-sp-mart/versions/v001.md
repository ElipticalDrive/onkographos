# USCG-based Sp-MART

Also called: USCG Sp-MART, Uniformly Sampled Cylindrical Grid Sp-MART

## Overview
USCG-based Sp-MART is a tomographic reconstruction method that reduces the space and time complexity of algebraic reconstruction techniques by discretizing the image domain using a Uniformly Sampled Cylindrical Grid (USCG) [S1]. It solves the radial deterioration of resolution found in conventional polar grid methods while accelerating line tracing by leveraging rotational symmetry and on-the-fly coefficient calculation [S1]. Compared to Cartesian grid-based methods, it enhances overall reconstruction speed by a factor of 2.5, and unlike the Feldkamp-Davis-Kress (FDK) algorithm, it produces high-quality images with a significantly limited number of projections and detectors, making it suitable for low-cost CT setups [S1].

## How it works
**Inputs:**
*   $p$: Number of projection views.
*   $d$: Number of detectors per projection.
*   $\theta_k$: Source angle for the $k$-th projection view ($k = 0, \dots, p-1$).
*   $P_k$: The $k$-th projection data vector of size $d$.
*   $V$: Initial volume estimate (initialized to zero or a prior guess).

**Outputs:**
*   $V$: Reconstructed volume.

**Data Structures:**
The USCG discretization organizes voxels into slices, rings, and angular sectors.
*   **Slice ($s$):** Index along the z-axis.
*   **Ring ($r$):** Radial index from the center.
*   **Angle ($\alpha$):** Angular position within the ring.
*   **Line Segment Attributes:** For each line segment in the first projection, store:
    *   $s$: Slice number.
    *   $r$: Ring number.
    *   $\alpha_0$: Angular position in the first projection ($k=0$).
    *   Intersection points with the grid boundaries (calculated only for one-fourth of the lines; others are mirrored) [S1].

**Parameters:**
*   Typical experimental setups use 50–70 projections and 10,000–40,000 detectors [S1].
*   X-ray voltages: 110–120 KeV [S1].
*   X-ray tube currents: 105–110 microamperes [S1].

**Algorithm Steps:**

1.  **Initialize Volume:** Set all voxel values in $V$ to zero or a background value.
2.  **First Projection Tracing ($k=0$):**
    *   Identify the lines corresponding to the first projection view.
    *   Due to symmetry, calculate intersection points and grid attributes ($s, r, \alpha_0$) for only one-fourth of the total lines [S1].
    *   Generate the remaining lines by mirroring the calculated intersection points [S1].
    *   Store the slice number ($s$), ring number ($r$), and initial angular position ($\alpha_0$) for each line segment [S1].
    *   Perform time-consuming steps (intersection calculation, array sorting, rounding, angular position finding) only for this projection [S1].
3.  **Iterative Update (Sp-MART):**
    For each iteration $iter$ and each projection view $k = 0, \dots, p-1$:
    *   **Trace Line for Projection $k$:**
        *   Retrieve the stored $s$ and $r$ for the line segment.
        *   Calculate the current angular position $\alpha_k$ using rotational symmetry:
            $$ \alpha_k = \alpha_0 + \theta_k $$
            where $\theta_k$ is the source angle of the $k$-th projection [S1].
        *   Note: The slice number $s$ and ring number $r$ remain unchanged from the first projection [S1].
    *   **Calculate Line Coefficients On-the-Fly:**
        *   Compute the path length $L$ of the line segment through the voxel defined by $(s, r, \alpha_k)$ [S1].
        *   Compute the projection coefficient $a_{ij}$ (weight of voxel $j$ in projection $i$) based on the intersection geometry. Since coefficients are calculated on the fly, no storage is required for the system matrix [S1].
    *   **Forward Projection:**
        *   Compute the estimated projection value $\hat{p}_i$ for detector $i$:
            $$ \hat{p}_i = \sum_{j \in \text{voxels}} a_{ij} V_j $$
    *   **Backprojection and Multiplicative Update:**
        *   Compute the ratio of measured to estimated projection:
            $$ R_i = \frac{p_i}{\hat{p}_i} $$
        *   Update the voxel values $V_j$ using the multiplicative ART rule:
            $$ V_j^{new} = V_j^{old} \times \left( \sum_{i \in \text{detectors}} a_{ij} R_i \right) / \left( \sum_{i \in \text{detectors}} a_{ij} \right) $$
            *Note: The specific normalization denominator $\sum a_{ij}$ ensures stability, consistent with standard MART formulations [G].*
4.  **Termination:** Repeat step 3 until convergence or a fixed number of iterations is reached.

**Edge Cases and Pitfalls:**
*   **Symmetry Assumption:** The method relies on the object and grid having rotational symmetry for the mirror-image optimization. If the object is highly asymmetric or the grid is not uniformly sampled, the one-fourth calculation may introduce artifacts [S1].
*   **Precision:** Rounding numbers during the initial trace of the first projection must be handled carefully to avoid accumulation errors in subsequent projections [S1].
*   **Degenerate Input:** Lines passing exactly through grid boundaries or the center axis require special handling for intersection calculations [S1].

## Operating characteristics
*   **Time Complexity:**
    *   Line tracing speed is enhanced by $O(N)$ compared to traditional polar grid methods because it avoids finding intersections with radial planes [S1].
    *   The line tracing process is accelerated by $O(p)$ where $p$ is the number of projection views, as time-consuming steps are executed only for the first projection [S1].
    *   Overall reconstruction speed is enhanced by a factor of 2.5 compared to Cartesian grid-based methods [S1].
*   **Memory Complexity:**
    *   Storage requirement is reduced by a factor of $p$ because projection coefficients are calculated on the fly and not stored [S1].
    *   Only the slice, ring, and angle for the first projection need to be stored [S1].
*   **Performance:**
    *   **Cu-Lump (50 projections, 10,000 detectors):** MAE 0.0011, RMSE 0.0037, SSIM 0.9909 [S1].
    *   **Frog (70 projections, 40,000 detectors):** MAE 0.0019, RMSE 0.0052, SSIM 0.9626 [S1].
    *   FDK requires 400 projections and >1 million detectors for comparable quality [S1].
*   **Strengths:**
    *   Ideal for low-cost CT setups with limited projections and detectors [S1].
    *   Solves radial deterioration of resolution inherent in conventional polar grids [S1].
*   **Limitations:**
    *   Relies on specific grid symmetries; less flexible for arbitrary Cartesian or non-uniform sampling without modification [S1].

## Background
The method was proposed to address the limitations of algebraic reconstruction techniques, specifically the high computational cost and memory usage associated with large system matrices in traditional polar or Cartesian grids [S1]. It builds on [Multiplicative ART](14391-multiplicative-art.md), which uses multiplicative updates for stability and positivity preservation. The USCG discretization scheme was introduced to solve the radial resolution deterioration found in conventional polar grid methods [S1]. Experimental validation was performed on a Copper Lump and a Frog specimen using X-ray voltages of 120 KeV and 110 KeV, respectively [S1].

## Sources
[S1] Uniformly Sampled Polar and Cylindrical Grid Approach for 2D, 3D Image Reconstruction using Algebraic Algorithm, https://arxiv.org/abs/2208.12964v1, fetched 2026-10-27
