# GPU Implementation of Precomputed Multiple Scattering

Also called: GPU lobe approximation

## Overview
This method adapts the Precomputed Multiple Scattering Table for efficient execution on graphics hardware by reducing memory footprint and enabling hierarchical storage. It solves the problem of high memory consumption and complex traversal required for full 5D multiple scattering data by approximating lobes with rotational symmetry and compressing spatial data. It differs from [Bounced Multiple Scattering](3532-bounced-multiple-scattering.md) by relying on precomputed lookup tables rather than runtime ray bouncing for the multiple scattering component.

## How it works
**Inputs and Outputs:**
*   **Input:** Precomputed multiple scattering data (albedo $\alpha$, phase function parameter $g$, spatial coordinates, angular directions).
*   **Output:** Illumination contribution for a pixel or sample point.

**Data Structures:**
1.  **5D Material Response Table:**
    *   Dimensions: $(\alpha, g, \rho/\ell, z/\ell, \theta)$.
    *   $\alpha$: Albedo.
    *   $g$: Phase function parameter.
    *   $\rho/\ell, z/\ell$: Normalized spatial coordinates relative to the source.
    *   $\theta$: Angle between the outgoing direction and the lobe axis.
    *   Total size: 600 MB [S1].
2.  **Illumination Sample Group:**
    *   Groups of 8 samples.
    *   Fields per sample: Position, Direction, Incoming Radiance, Scaled Surface Area, Interval Length along the ray [S1].

**Steps:**
1.  **Lobe Approximation:** Approximate each full scattering lobe with a lobe possessing symmetry of revolution around the axis joining the source point $O$ and the current point $P$. This reduces the angular dimension from full spherical coordinates to a single angle $\theta$ [S1].
2.  **Table Construction:** Compute material responses for multiple discrete values of $\alpha$ and $g$. Store these in the 5D table indexed by $(\rho/\ell, z/\ell, \theta)$ [S1].
3.  **Hierarchical Storage:** Store illumination samples in a spatial hierarchy (e.g., octree or similar structure implied by "hierarchical data structure storage") to optimize traversal [S1]. Group individual samples into sets of eight for efficient memory access [S1].
4.  **Runtime Interpolation:** For a given material with specific $\alpha$ and $g$, interpolate within the 5D table to extract the corresponding 3D response table $(\rho/\ell, z/\ell, \theta)$ [S1].
5.  **Rendering:** Traverse the spatial hierarchy to retrieve relevant illumination samples. Compute the contribution using the interpolated material response and the stored sample properties (position, direction, radiance, area, interval) [S1].

**Parameters:**
*   Table Size: 600 MB [S1].
*   Group Size: 8 samples per group [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The approximation of full lobes to symmetric lobes causes a small decrease in Peak Signal to Noise Ratio (PSNR), though the visual impact is negligible [S2].
*   **Memory:** The 5D table requires significant GPU memory (600 MB), necessitating careful management or hierarchical compression [S1].

## Subtopics

### Precomputed Multiple Scattering with Quadtree Compression
A variant that stores precomputed multiple scattering in a table compressed with quadtrees for adaptive representation, decoupling double and multiple scattering. It uses the same spatial dimension (2) but one less angular dimension compared to concentric spheres methods. This approach provides better approximation, especially near the propagation axis, and avoids Mach banding effects caused by insufficient samples in concentric sphere methods [S2]. It is used in UPBP (Unidirectional Path Beamer), VRL (Virtual Ray Lights), and MEMLT (Metropolis Light Transport) to accelerate rendering [S2].

## Operating characteristics
*   **Memory:** The base 5D table requires 600 MB [S1]. Quadtree compression offers adaptive representation with equal memory cost to concentric spheres but better approximation [S2].
*   **Performance:** Significant reduction in time spent per interaction allows rendering more iterations (UPBP), more virtual ray lights (VRL), or more samples (MEMLT) within the required time frame [S2].
*   **Quality:** Visual quality is close to reference images and better than unmodified algorithms for small computation times (higher PSNR) [S2]. Works well across the entire mean-free-path (mfp) range, whereas other methods work better only for smaller mfp (opaque materials) [S2].
*   **Limitations:** The lobe symmetry approximation introduces a small PSNR decrease [S2].

## Background
This method was developed to adapt precomputed multiple scattering for GPU-specific constraints, including hierarchical data structure storage and traversal [S1]. It builds on the work of Moon et al. [20] (cited in [S2]) regarding shell tracing and concentric spheres. The quadtree compression variant was introduced to improve approximation quality and avoid artifacts associated with concentric spheres [S2].

## Sources
[S1] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
[S2] Precomputed Multiple Scattering for Rapid Light Simulation in Participating Media, https://doi.org/10.1109/TVCG.2018.2890466, fetched 2026-10-26
