# Precomputed Multiple Scattering Table

Also called: PMS, Precomputed Multiple Scattering, Multiple scattering precomputation, Multiple Scattering by Precomputation, Precomputed Multiple Scattering Simulation, Precomputed Multiple Scattering for Participating Media, Precomputed multiple scattering for high-order scattering, Precomputed Multiple Scattering for Rapid Light Simulation

## Overview
Precomputed Multiple Scattering Table (PMS) solves the slow convergence of high-order scattering media rendering by precomputing multiple scattering effects in an infinite participating medium [S1, S2]. It replaces the runtime computation of multiple scattering events with queries to compact precomputed tables, allowing existing global illumination algorithms (such as VRL, UPBP, or MEMLT) to handle single and double scattering while combining their results with the precomputed multiple scattering contribution [S2]. Unlike [Beam Spreading Width Model](1193-beam-spreading-width-model.md) or [PSF-based Multiple Scattering Rendering](1297-psf-based-multiple-scattering-rendering.md) which approximate scattering profiles, PMS explicitly simulates photon transport in an infinite medium to capture complex angular distributions [S2].

## How it works
**Inputs:** Material properties (index of refraction $\eta$, scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, phase function $p(\omega, \omega_t)$), light source geometry (point or segment), and scene geometry for integration with a base algorithm.
**Outputs:** A precomputed table (or neural network representation) of multiple scattering radiance, and the final rendered image where multiple scattering is added to low-order scattering computed by the base algorithm.

**Data Structures:**
1.  **Precomputed Table:** A 4D representation of outgoing radiance $L_o(\rho/\ell, z/\ell, \theta, \phi)$.
    *   Spatial dimensions: Cylindrical coordinates $(\rho, z)$ normalized by the mean-free-path $\ell$.
    *   Angular dimensions: Spherical coordinates $(\theta, \phi)$ relative to the local frame defined by the light direction.
    *   Storage: Compressed using a quadtree structure for spatial subdivision and lobe approximation for angular data [S2, S3].
2.  **Quadtree:**
    *   Root: $32 \times 32$ regular grid [S2].
    *   Subdivision: Cells subdivide when photon count exceeds a threshold [S2].
    *   Leaf: Stores lobe approximation data [S2].
3.  **Lobe Approximation:**
    *   For point sources: Axial symmetry approximation. Memory: 55 floats per lobe [S2].
    *   For segment sources: Asymmetric lobe approximation. Memory: 91 floats per lobe [S2].
    *   Angular sampling: $\theta$ sampled every $10^\circ$ (18 samples), $\phi$ sampled with 36 directions [S3].

**Parameters:**
*   Mean-free-path $\ell = 1/\sigma_t$, where $\sigma_t = \sigma_s + \sigma_a$ [S2].
*   Spatial range: $\rho$ and $z$ from 0 to 32 mfp; up to 64 mfp for highly anisotropic materials with large $g$ [S2].
*   Segment source length: $2 \times \text{mfp}$ [S2].

**Steps:**

### Precomputation Stage
1.  **Assume Infinite Medium:** Define an infinite, homogeneous participating medium with the target material properties ($\eta, \sigma_s, \sigma_a, p$) [S2].
2.  **Define Light Sources:**
    *   **Point Source:** Directional light source shooting photons along the z-axis [S2, S3].
    *   **Segment Source:** Defined by origin $o=(0,0)$, direction $v$ aligned with z-axis, and length $l=2 \times \text{mfp}$ [S2].
3.  **Photon Shooting:** Shoot photons from the light source into the infinite medium. Particles propagate in straight lines, scatter, and absorb using standard Monte-Carlo procedures [S3].
4.  **Normalization:** Normalize all spatial coordinates by the mean-free-path $\ell$ to reduce parameter dependence [S2, S3].
5.  **Accumulation:** Accumulate radiance contributions into a 4D table.
    *   Spatial indexing: Cylindrical coordinates $(\rho/\ell, z/\ell)$ [S3].
    *   Angular indexing: Spherical coordinates $(\theta, \phi)$ relative to the propagation direction [S3].
    *   Each grid cell acts as a kernel gathering particles scattered inside it [S2].
6.  **Compression:**
    *   **Spatial:** Build a quadtree. Start with a $32 \times 32$ grid. Accumulate photons. Subdivide cells when photon count exceeds a threshold. Discard nodes with no scattering events [S2, S3].
    *   **Angular:** Approximate lobes. For point sources, use axial symmetry. For segment sources, use asymmetric lobes. Sample $\theta$ every $10^\circ$ [S2].

### Rendering Stage (Integration with Base Algorithm)
The base algorithm (e.g., VRL, UPBP, MEMLT) computes single and double scattering. PMS handles multiple scattering.

**For MEMLT Integration:**
1.  **Path Division:** Divide a seed path $\bar{x}$ (with at least three volume scattering events) into three subpaths: $\bar{x}_1$ (light source to first volume event), $\bar{x}_2$ (first to last volume event), and $\bar{x}_3$ (last volume event to camera) [S2].
2.  **Special Edge:** Replace subpath $\bar{x}_2$ with a special edge transporting multiple scattering between its endpoints $x_i$ and $x_o$ [S2].
3.  **Table Query:** Compute the multiple scattering contribution $mul(x_i, \omega_i, x_o, \omega_o)$ using the precomputed table [S2].
    *   Calculate cylindrical coordinates of the target point $P_k$ around the axis defined by the source vertex $v_i$ and direction $d_i$:
        $$ z_k = |\mathbf{r}_i \cdot \hat{\mathbf{d}}_i| $$
        $$ \rho_k = \|\mathbf{r}_i - z_k \hat{\mathbf{d}}_i\| $$
    *   Extract radiance from the table $L_0$:
        $$ \text{mult.}(P_k) = \sum_{i_1} L_0 \left( \frac{\rho_k}{\ell}, \frac{z_k}{\ell}, T_{(i_1, \hat{d}_1)}(\hat{\mathbf{o}}) \right) $$
        where $T_{(i_1, \hat{d}_1)}(\hat{\mathbf{o}})$ transforms the outgoing direction $\hat{\mathbf{o}}$ into the frame defined by $(\hat{v}_i, \hat{d}_i)$ [S3].
4.  **Mutation Rules:**
    *   Use the table value as the weight for $\bar{x}_2$.
    *   Set the probability density function (pdf) of the special edge to 1 [S2].
    *   Consider both endpoints as non-specular [S2].
    *   Prevent the endpoint connected to the light source from being a mutation source [S2].
    *   Only surface events and the endpoints of the special path may be mutated during path construction [S1].

**For VRL Integration:**
1.  **Phase 1:** Store only single scattering events in the volume (allowing multiple bounces) [S2].
2.  **Phase 2:** Compute double scattering using importance sampling and multiple scattering using the precomputed tables [S2]. Connect point/segment from light side and point/segment from camera side directly using the tables [S2].

**For UPBP Integration:**
1.  **Photon Shooting:** Restrict scattering to at most two events. Store single-scattered and double-scattered beams separately [S2].
2.  **Gathering:** Use double-scattered beams for double scattering. Use single-scattered beams for single scattering and multiple scattering via precomputed tables [S2].

**Pitfalls and Edge Cases:**
*   **Bias:** The method introduces bias because the special path contribution is based on density estimation in the precomputation pass [S1].
*   **Visibility:** The method does not consider visibility between the source and query or boundaries. This causes inaccuracy in scenes with occlusion or air inside the medium [S2].
*   **Infinite Medium Assumption:** The table assumes an infinite medium. It does not account for light bouncing off internal surfaces [S1]. This requires correction via [Bounced Multiple Scattering](3532-bounced-multiple-scattering.md) if internal reflections are significant.

## Subtopics

### [Bounced Multiple Scattering](3532-bounced-multiple-scattering.md)
Bounced Multiple Scattering corrects for light that has bounced off the internal surface of a material, which is missing from the infinite medium precomputed multiple scattering table. It adds diffuse indirect illumination from inside the material to surface samples, allowing the renderer to account for light that enters the volume, scatters, hits the boundary, reflects back into the volume, and scatters again toward the camera. It is a correction term applied after the primary multiple scattering lookup, distinct from single and double scattering approximations.

### [GPU Implementation of Precomputed Multiple Scattering](4706-gpu-implementation-of-precomputed-multiple-scattering.md)
This method adapts the Precomputed Multiple Scattering Table for efficient execution on graphics hardware by reducing memory footprint and enabling hierarchical storage. It solves the problem of high memory consumption and complex traversal required for full 5D multiple scattering data by approximating lobes with rotational symmetry and compressing spatial data. It differs from [Bounced Multiple Scattering](3532-bounced-multiple-scattering.md) by relying on precomputed lookup tables rather than runtime ray bouncing for the multiple scattering component.

### Neural Network Multiple Scattering Representation
Uses neural networks instead of tables to represent multiple scattering, reducing memory usage from hundreds of MB to kB without significant reduction in quality [S1]. It maps a seven-dimensional function (anisotropy $g$, albedo $\alpha$, location $r(\rho, z)$, direction $\theta, \phi$) to radiance via a four-layer neural network [S1]. Storage is reduced from 50 GB to 23.6 kB for double and multiple scattering [S1]. During rendering, the network reproduces the precomputed table for a specific medium, which is then used for multiple scattering computation [S1].

## Operating characteristics
*   **Time Complexity:** Precomputation is expensive (Monte-Carlo simulation in infinite medium), but rendering is fast. For a 512x512 image of a BumpS. scene, VRL with PMS took 53s vs 1s for VRLs alone (PSNR 34.5 vs 23.21) [S2]. For a Wax scene, 180s with VRL vs 40s for VRLs alone (PSNR 37.4 vs 12.28) [S2].
*   **Memory Complexity:** The hierarchical representation reduces memory cost from 8.4 GB to approximately 60 MB [S3]. Lobe approximation adds 55 floats (point) or 91 floats (segment) per lobe [S2]. Neural network variant reduces this to 23.6 kB [S1].
*   **Strengths:** Significantly faster convergence for high-order scattering media. Negligible impact on accuracy compared to fully converged solutions. Easy integration with existing algorithms (VRL, UPBP, MEMLT) [S2].
*   **Limitations:** Introduces bias. Does not handle visibility or boundaries (inaccurate with occlusion/air). Assumes homogeneous, infinite medium [S2].
*   **When to use:** Scenes with high albedo or small mean-free-path where high-order scattering dominates and convergence is slow [S2].
*   **When not to use:** Scenes with significant internal reflections (requires Bounced Multiple Scattering correction) or complex visibility/occlusion within the medium [S2].

## Background
The method was introduced by Wang et al. [23] to solve the slow convergence of high-order scattering media rendering [S1]. It was further developed and integrated with VRL, UPBP, and MEMLT by Ge et al. [S2]. The approach exploits the rotational symmetry of multiple scattering in infinite homogeneous media to compress the data into compact tables [S2, S3]. Later, Ge et al. [65] proposed using neural networks to replace the tables, further reducing memory usage [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Precomputed Multiple Scattering for Rapid Light Simulation in Participating Media, https://doi.org/10.1109/TVCG.2018.2890466, fetched 2026-10-26
[S3] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
