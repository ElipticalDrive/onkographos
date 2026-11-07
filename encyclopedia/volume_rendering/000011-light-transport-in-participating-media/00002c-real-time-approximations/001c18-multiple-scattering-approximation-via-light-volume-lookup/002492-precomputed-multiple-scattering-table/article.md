# Precomputed Multiple Scattering Table

Also called: PMS

## Overview
Precomputed Multiple Scattering Table (PMS) accelerates light simulation in participating media by precomputing the radiance contribution of multiple scattering events (more than two scattering events) in an infinite homogeneous medium, storing the results in compact 4D lookup tables [S2, S3]. It solves the slow convergence of high-order scattering rendering by allowing existing global illumination algorithms (such as VRL, UPBP, and MEMLT) to handle single and double scattering explicitly, while querying the precomputed tables for the remaining multiple scattering contribution [S1, S2]. Unlike [Beam Spreading Width Model](1193-beam-spreading-width-model.md) which approximates scattering via geometric width, or [PSF-based Multiple Scattering Rendering](1297-psf-based-multiple-scattering-rendering.md) which uses spatial filters, PMS relies on Monte-Carlo precomputation of directional lobes [S2, S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Material properties (scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, phase function $p(\omega, \omega_t)$, index of refraction $\eta$); Light source geometry (point directional or segment); Rendering algorithm context (e.g., VRL rays, UPBP beams, MEMLT seed paths).
*   **Outputs:** A compressed quadtree data structure containing 4D radiance tables (two tables: one for point sources, one for segment sources); Final pixel radiance combining low-order scattering from the host algorithm and multiple scattering from the table.

**Data Structures:**
The core data structure is a quadtree representing a 2D spatial grid in cylindrical coordinates $(\rho/\ell, z/\ell)$, where $\ell$ is the mean-free path.
*   **Spatial Domain:** $\rho$ (radial distance from propagation axis) and $z$ (depth along propagation axis). Coordinates are normalized by $\ell$. The grid typically spans $0$ to $32$ mfp, extending to $64$ mfp for highly anisotropic materials [S3].
*   **Angular Domain:** At each spatial node, outgoing radiance is stored as a "lobe" in spherical coordinates $(\theta, \phi)$ relative to the local propagation frame [S2, S3].
*   **Lobe Representation:**
    *   **Point Source:** Exploits axial symmetry. The lobe is approximated by sampling $\theta$ every $10^\circ$ (18 samples). Memory cost: 55 floats per lobe [S3].
    *   **Segment Source:** Asymmetric lobes. Memory cost: 91 floats per lobe [S3].
*   **Quadtree:** Begins as a $32 \times 32$ regular grid. Cells are subdivided when a photon count threshold is reached. Nodes with no scattering events are discarded [S3].

**Precomputation Steps:**
1.  **Define Infinite Medium:** Assume an infinite, homogeneous medium with the target material properties ($\sigma_s, \sigma_a, p, \eta$). Calculate mean-free path $\ell = 1/(\sigma_s + \sigma_a)$ [S3].
2.  **Define Light Sources:**
    *   **Point Source:** Directional light shooting along the z-axis.
    *   **Segment Source:** Defined by origin $o=(0,0)$, direction $v$ aligned with z-axis, and length $l = 2 \times \text{mfp}$ [S3].
3.  **Photon Shooting:** Shoot photons from the light source into the infinite medium. Propagate photons using standard Monte-Carlo procedures (scattering and absorption) [S2, S4].
4.  **Accumulation:** For each photon, determine its position $(\rho, z)$ relative to the source axis. Accumulate radiance contributions into the corresponding spatial grid cell. The radiance is evaluated in spherical coordinates $(\theta, \phi)$ relative to the current frame [S3].
5.  **Spatial Subdivision:** After an initial pass, subdivide grid cells based on photon density to create the quadtree structure [S3].
6.  **Lobe Approximation:** Inside each final quadtree node, approximate the accumulated directional distribution. For point sources, fit the axial symmetry; for segment sources, store the asymmetric lobe [S3].
7.  **Compression:** Compress the quadtree by removing empty nodes. Total precomputation time is approx. 1 minute for high-albedo materials; storage cost is approx. 100 MB uncompressed, reduced to ~60 MB with hierarchical compression [S2, S4].

**Rendering Integration:**
The method integrates with host algorithms by replacing complex sub-paths with a "special edge" whose weight is retrieved from the table.

*   **For VRL (Virtual Ray Lights):**
    1.  Shoot photons in the scene, storing only the first scattering event in the volume (single scattering) [S2, S3].
    2.  During rendering, use these virtual rays (segments) as the source.
    3.  Compute double scattering via importance sampling.
    4.  Compute multiple scattering by querying the segment-source table using the virtual ray as the source and the camera ray as the target [S2, S3].

*   **For UPBP (Unified Point Beams and Paths):**
    1.  Store first and second scattering events (beams/points) for single and double scattering [S2].
    2.  Use single-scattered beams as the source for multiple scattering.
    3.  Query the table to compute multiple scattering contributions from these beams to the camera rays [S2, S3].

*   **For MEMLT (Manifold Exploration Metropolis Light Transport):**
    1.  Identify a seed path with at least three volume scattering events.
    2.  Divide the path into three sub-paths: $\bar{x}_1$ (light to first volume event), $\bar{x}_2$ (first to last volume event), $\bar{x}_3$ (last volume event to camera) [S3].
    3.  Replace sub-path $\bar{x}_2$ with a special edge connecting the first and last volume events.
    4.  The weight of this edge is the multiple scattering contribution $mul(x_2, \omega_1, x_{k-2}, \omega_{k-2})$ retrieved from the table [S3].
    5.  **Mutation Rules:** Both endpoints of the special edge are treated as non-specular. The endpoint from the light source cannot be a mutation source. The PDF for the special edge is set to 1 [S1, S3].

**Lookup Formula:**
To compute multiple scattering from a source vertex $v_i$ with direction $\hat{d}_i$ to a target point $P_k$:
1.  Calculate cylindrical coordinates of $P_k$ relative to the axis $(v_i, \hat{d}_i)$:
    $$ z_k = |\mathbf{r}_i \cdot \hat{\mathbf{d}}_i| $$
    $$ \rho_k = \|\mathbf{r}_i - z_k \hat{\mathbf{d}}_i\| $$
2.  Normalize by mean-free path $\ell$.
3.  Transform the outgoing direction $\hat{o}$ into the local frame defined by $(v_i, \hat{d}_i)$ using rotation $T_{(i_1, \hat{d}_1)}(\hat{o})$.
4.  Query the table:
    $$ \text{mult.}(P_k) = \sum_{i_1} L_0 \left( \frac{\rho_k}{\ell}, \frac{z_k}{\ell}, T_{(i_1, \hat{d}_1)}(\hat{o}) \right) $$
    where $L_0$ is the stored radiance in the table [S4].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def precompute_table(material, source_type, num_photons):
    """
    Precomputes the multiple scattering table for an infinite medium.
    material: {sigma_s, sigma_a, phase_function, eta}
    source_type: 'point' or 'segment'
    num_photons: int
    Returns: Quadtree structure
    """
    sigma_t = material.sigma_s + material.sigma_a
    l = 1.0 / sigma_t  # Mean-free path
    
    # Initialize spatial grid (32x32)
    grid = QuadtreeGrid(resolution=32, max_depth=5)
    
    for _ in range(num_photons):
        # Shoot photon from source
        if source_type == 'point':
            pos = (0, 0, 0)
            dir = (0, 0, 1) # Along z-axis
        else: # segment
            # Segment from (0,0,0) to (0,0, 2*l)
            # Sample point on segment
            t = random.uniform(0, 2 * l)
            pos = (0, 0, t)
            dir = (0, 0, 1)
            
        # Simulate propagation in infinite medium
        current_pos = pos
        current_dir = dir
        throughput = 1.0
        
        while throughput > 0.001:
            # Sample distance to next interaction
            dist = -log(random.uniform(0, 1)) / sigma_t
            
            # Move photon
            next_pos = current_pos + current_dir * dist
            
            # Determine spatial coordinates relative to source axis (z-axis)
            # rho is distance from z-axis, z is depth along z-axis
            rho = sqrt(next_pos.x**2 + next_pos.y**2)
            z = next_pos.z
            
            # Normalize by mfp
            rho_norm = rho / l
            z_norm = z / l
            
            # Accumulate contribution to grid cell at (rho_norm, z_norm)
            # Direction is current_dir
            grid.accumulate(rho_norm, z_norm, current_dir, throughput)
            
            # Scatter
            new_dir = sample_phase_function(material.phase_function, current_dir)
            albedo = material.sigma_s / sigma_t
            if random.uniform(0, 1) < albedo:
                throughput *= albedo
                current_dir = new_dir
                current_pos = next_pos
            else:
                break
                
    # Compress grid into quadtree
    compressed_tree = grid.compress()
    return compressed_tree

def query_multiple_scattering(source_vertex, source_dir, target_point, table, material):
    """
    Queries the precomputed table for multiple scattering contribution.
    """
    sigma_t = material.sigma_s + material.sigma_a
    l = 1.0 / sigma_t
    
    # Vector from source to target
    r = target_point - source_vertex
    
    # Cylindrical coordinates relative to source_dir axis
    # z_k is projection of r onto source_dir
    z_k = dot(r, source_dir)
    # rho_k is perpendicular distance
    projection = source_dir * z_k
    rho_k = norm(r - projection)
    
    # Normalize
    rho_norm = rho_k / l
    z_norm = z_k / l
    
    # Determine outgoing direction from target to camera (or next vertex)
    # Assuming 'outgoing_dir' is provided by the rendering context
    # Transform outgoing_dir to local frame of (source_vertex, source_dir)
    local_outgoing_dir = transform_to_local_frame(outgoing_dir, source_vertex, source_dir)
    
    # Query table
    # The table stores radiance L0(rho_norm, z_norm, theta, phi)
    # Interpolate or sample from the lobe at (rho_norm, z_norm)
    radiance = table.query(rho_norm, z_norm, local_outgoing_dir)
    
    return radiance
```

**Parameters:**
*   **Grid Resolution:** $32 \times 32$ initial grid [S3].
*   **Spatial Range:** $0$ to $32$ mfp (up to $64$ mfp for high anisotropy $g$) [S3].
*   **Angular Sampling:** $\theta$ sampled every $10^\circ$ (18 samples) [S3].
*   **Segment Length:** $2 \times \text{mfp}$ [S3].

**Edge Cases and Pitfalls:**
*   **Bias:** The method introduces bias because the special path contribution is based on density estimation in the precomputation pass [S1]. Bias can be reduced by letting the host algorithm handle more low-order scattering events (3-4 events) [S2].
*   **Visibility:** The method does not account for visibility between the source and query or boundaries. This causes inaccuracy in scenes with occlusion or air inside the medium [S3].
*   **Material Dependence:** The table depends only on albedo and phase function. Different materials require different tables [S4].

## Subtopics

### [Bounced Multiple Scattering](3532-bounced-multiple-scattering.md)
Bounced Multiple Scattering corrects for light that has bounced off the internal surface of a material, which is missing from the infinite medium precomputed multiple scattering table. It adds diffuse indirect illumination from inside the material to surface samples, allowing the renderer to account for light that enters the volume, scatters, hits the boundary, reflects back into the volume, and scatters again toward the camera. It is a correction term applied after the primary multiple scattering lookup, distinct from single and double scattering approximations.

### [GPU Implementation of Precomputed Multiple Scattering](4706-gpu-implementation-of-precomputed-multiple-scattering.md)
This method adapts the Precomputed Multiple Scattering Table for efficient execution on graphics hardware by reducing memory footprint and enabling hierarchical storage. It solves the problem of high memory consumption and complex traversal required for full 5D multiple scattering data by approximating lobes with rotational symmetry and compressing spatial data. It differs from [Bounced Multiple Scattering](3532-bounced-multiple-scattering.md) by relying on precomputed lookup tables rather than runtime ray bouncing for the multiple scattering component.

### [Multiple Scattering Precomputation](9418-multiple-scattering-precomputation.md)
This method computes the radiative transfer equation for participating media by splitting it into incremental terms for $K$ scattering orders, allowing precomputation of multiple scattering effects that would otherwise require exponentially complex recursive Monte-Carlo evaluation. It differs from [Bounced Multiple Scattering](3532-bounced-multiple-scattering.md) by using explicit integral precomputation of auxiliary fields rather than ray-bouncing, and from [GPU Implementation of Precomputed Multiple Scattering](4706-gpu-implementation-of-precomputed-multiple-scattering.md) by focusing on the mathematical formulation and data layout rather than hardware-specific optimization.

### Neural Network Multiple Scattering Representation
Uses neural networks instead of tables to represent multiple scattering, reducing memory usage from hundreds of MB to kB without significant reduction in quality [S1]. It maps a seven-dimensional function (anisotropy $g$, albedo $\alpha$, location $r(\rho, z)$, direction $\theta, \phi$) to radiance via a four-layer neural network [S1]. During rendering, the network reproduces the precomputed table for a specific medium, which is then used for multiple scattering computation [S1]. Storage is reduced from 50 GB to 23.6 kB [S1].

## Operating characteristics
*   **Time Complexity:** Precomputation time is approx. 1 minute for high-albedo materials [S2]. Rendering time is significantly reduced compared to full path tracing. For example, VRL with PMS took 53s vs 1s for VRL alone (but VRL alone had much lower PSNR) [S3].
*   **Memory Complexity:** Storage cost after compression is around 100 MB for the base method [S2]. Hierarchical representation reduces this to approx. 60 MB [S4]. Neural network variants reduce this to 23.6 kB [S1].
*   **Performance:** On a 2.20GHz Intel i7 (40 cores), for a 512x512 image of a BumpS scene, VRL with PMS took 53s (PSNR 34.5) vs 1s for VRL alone (PSNR 23.21) [S3]. For a Wax scene, 180s (PSNR 37.4) vs 40s (PSNR 12.28) [S3].
*   **Strengths:** Greatly reduces convergence time for high-order scattering media; integrates easily with existing algorithms (VRL, UPBP, MEMLT); higher spatial and angular resolution than concentric spheres methods [S2, S3].
*   **Limitations:** Introduces bias; does not handle visibility or boundaries; requires precomputation per material; less effective for low-albedo materials where multiple scattering is negligible [S1, S2, S3].

## Background
Originated by Beibei Wang and Nicolas Holzschuch in 2017 at SIGGRAPH 2017 Talks, extending Wang et al. [2016] by adding multiple scattering from a segment instead of a point [S2]. It was implemented in the Mitsuba Renderer [Jakob 2010] for VRL and MEMLT, and in smallUPBP [Křivánek 2014] for UPBP [S2, S3].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Precomputed Multiple Scattering for Light Simulation in Participating Medium, https://doi.org/10.1145/3084363.3085037, fetched 2026-10-26
[S3] Precomputed Multiple Scattering for Rapid Light Simulation in Participating Media, https://doi.org/10.1109/TVCG.2018.2890466, fetched 2026-10-26
[S4] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
