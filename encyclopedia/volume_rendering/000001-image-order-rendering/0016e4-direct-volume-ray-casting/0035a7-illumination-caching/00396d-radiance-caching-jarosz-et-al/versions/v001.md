# Radiance Caching (Jarosz et al.)
Also called: Jarosz Radiance Caching

## Overview
Radiance caching is an illumination caching technique that pre-computes and stores directional radiance distributions at sparse points within a volume, using spherical harmonics (SH) to represent the data [S1]. It solves the problem of expensive per-pixel lighting calculations in participating media by interpolating illumination from these cached samples, thereby reducing computational load and providing smoother results [S1]. It differs from sibling methods like the Light Cache Rendering Pipeline by explicitly modeling the angular distribution of light via SH coefficients rather than storing scalar irradiance or simple light maps [S1].

## How it works
**Inputs:** Sparse set of pre-computed sample points within the volume, each associated with a stored SH coefficient vector representing the incoming radiance distribution and its gradient [S1].
**Outputs:** Interpolated illumination (radiance) at arbitrary query points within the volume [S1].

**Data Structures:**
*   **Cache Point:** A spatial location $(x, y, z)$ in the volume.
*   **SH Coefficients:** A vector of coefficients $\lambda_l^m$ representing the radiance distribution $L_i(\theta, \phi)$ at that point. For a 2nd-order SH representation, this typically involves 4 coefficients (mapping to RGB or similar component buffers) [S2, unofficial].
*   **Gradient Information:** Derivative data associated with the cache points to facilitate interpolation [S1].

**Steps:**
1.  **Pre-computation (Implicit):** Radiance distributions are computed at sparse points. In related implementations (e.g., Volume Sampling and Radiance Caching), this involves ray-marching through the volume to determine incoming radiance $L_i$ for various directions and projecting it onto SH basis functions [S2, unofficial].
2.  **Query:** Given a new position $P$ in the volume where illumination is required:
    a. Identify the nearest cache points or the relevant neighborhood in the sparse grid [S1].
    b. Retrieve the stored SH coefficients and gradient information for these points [S1].
    c. Interpolate the SH coefficients and/or gradients from the neighbors to $P$ [S1].
    d. Evaluate the SH function at $P$ using the interpolated coefficients to obtain the directional radiance distribution [S1].
    e. Integrate or sample from this distribution to get the final illumination value [S1].

**Formulas:**
The radiance distribution $L_i(\theta, \phi)$ at a cache point is represented as:
$$ L_i(\theta, \phi) = \sum_{l=0}^{n-1} \sum_{m=-l}^{l} \lambda_l^m Y_{lm}(\theta, \phi) $$
where $n$ is the order of the SH representation, $Y_{lm}$ are the spherical harmonic basis functions, and $\lambda_l^m$ are the coefficients [S2, unofficial].

The coefficients are computed via Monte Carlo quadrature with uniform sampling [S2, unofficial]:
$$ \lambda_l^m = \frac{4\pi}{N} \sum_{j=1}^{N} L_i(\theta_j, \phi_j) Y_{lm}(\theta_j, \phi_j) $$
where $N$ is the total number of samples, and $(\theta_j, \phi_j)$ are the sampled directions [S2, unofficial].

**Parameters:**
*   **SH Order ($n$):** Typically 2nd order is used to map efficiently to four-component graphics hardware buffers [S2, unofficial].
*   **Sample Count ($N$):** Number of random directions used for Monte Carlo integration. Specific values are not given in the primary source, but related implementations use stratified sampling [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Occlusion Mismatch:** Since stored sample points are sparse, a query point may be occluded differently than the nearest cache points. The original method by Jarosz et al. did not consider visibility during interpolation, leading to incorrect illumination in such cases [S1].
*   **Interpolation Error:** Simple interpolation of SH coefficients can introduce artifacts if the underlying radiance field changes rapidly or if occlusion boundaries are crossed [S1].

## Subtopics

### Volume Sampling and Radiance Caching
This variant explicitly computes the incoming radiance distribution for each voxel by ray-marching through the volume and storing the result as spherical harmonic coefficients [S2, unofficial]. It differs from the general Jarosz method by defining a voxel-based grid for caching rather than arbitrary sparse points [S2, unofficial]. It is chosen when a structured voxel representation is already available or when integrating with deferred rendering pipelines that update voxels based on visible surface reprojection [S2, unofficial]. The implementation uses 2nd-order SH (4 coefficients) to fit into standard 4-component framebuffers [S2, unofficial]. Coefficients are computed using Monte Carlo quadrature with stratified sampling of $N$ random directions per voxel [S2, unofficial].

## Operating characteristics
**Time Complexity:** Depends on the density of cache points and the cost of interpolation. Pre-computation is expensive due to ray-marching for each sample point/voxel [S2, unofficial]. Query time is fast, involving interpolation and SH evaluation [S1].
**Memory Complexity:** Proportional to the number of cache points/voxels times the number of SH coefficients per point. 2nd-order SH requires 4 coefficients per point [S2, unofficial].
**Measured Performance:** In a related implementation (Volume Sampling and Radiance Caching), ray-marching times for GI per bounce ranged from 4.3 to 4.6 ms on a NVIDIA GTX460 for scenes with 10k-260k triangles, while AO was faster due to limited ray-marching distance [S2, unofficial].
**Strengths:** Reduces per-pixel lighting calculations; provides smoother results than unfiltered sampling [S1].
**Limitations:** Sparse sampling can lead to visibility errors if occlusion is not considered during interpolation [S1]. Quality degrades in regions with high-frequency lighting changes or sharp occlusion boundaries [S1].
**When to use:** Suitable for scenes with smooth lighting variations where pre-computation cost is acceptable and high-frequency shadowing is not critical or can be handled by other means.
**When not to use:** Scenes with complex, dynamic occlusion or high-frequency lighting details where the sparse cache cannot accurately represent the radiance field [S1].

## Background
Radiance caching was introduced by Jarosz et al. [S1]. It is part of the broader field of illumination caching for volume rendering. The method was later improved by Marco et al. [S1], who introduced a second-order Hessian error metric to determine interpolation acceptability and consider visibility, resulting in higher quality than the original Jarosz method [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2011/33186/, fetched 2026-10-29 (unofficial)
