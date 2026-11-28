# Precomputed Multiple Scattering Table

Also called: PMS

## Overview
Precomputed Multiple Scattering Table (PMS) accelerates light transport in participating media by replacing the slow convergence of high-order scattering simulations with precomputed radiance data from an infinite homogeneous medium [S1, S2]. It solves the problem of expensive Monte Carlo convergence for multiple scattering events by storing the material response in compact 4D tables, allowing existing global illumination algorithms (such as VRL, UPBP, and MEMLT) to handle single and double scattering while querying the table for higher-order effects [S2, S3]. Unlike [Beam Spreading Width Model](1193-beam-spreading-width-model.md) which approximates the shape of the light beam, PMS stores the full angular radiance distribution (lobes) [S3]. It differs from [Neural Network Multiple Scattering Representation](7796-neural-network-multiple-scattering-representation.md) by using discrete lookup tables rather than learned regression functions [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Material properties (index of refraction $\eta$, scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, phase function $p(\omega, \omega_t)$); Light source geometry (point directional or segment); Scene geometry (for integration with specific algorithms).
*   **Outputs:** Two 4D precomputed tables (one for point source, one for segment source) containing outgoing radiance lobes; Rendered image with multiple scattering contributions.

**Data Structures:**
The method uses two 4D tables indexed by spatial coordinates $(\rho, z)$ and angular coordinates $(\theta, \phi)$ [S2, S3].
1.  **Spatial Domain:** Cylindrical coordinates $(\rho, z)$ normalized by the mean-free-path $\ell = 1/(\sigma_s + \sigma_a)$ [S3, S4].
    *   $\rho$: Radial distance from the propagation axis.
    *   $z$: Depth along the propagation axis.
    *   Range: $0$ to $32$ mfp for standard materials; up to $64$ mfp for highly anisotropic materials [S3].
2.  **Angular Domain:** Spherical coordinates $(\theta, \phi)$ relative to the local frame defined by the light direction [S3, S4].
    *   $\theta$: Polar angle, sampled every $10^\circ$ (18 samples) [S3].
    *   $\phi$: Azimuthal angle, sampled with 36 directions [S4].
3.  **Storage Layout:**
    *   **Raw Grid:** A $1024 \times 512$ 2D grid for spatial coordinates [S4].
    *   **Compression:** A quadtree structure subdivides the spatial grid based on photon density [S3, S4]. Nodes with no scattering events are discarded [S4].
    *   **Lobe Representation:** Lobes are computed inside each cell of the quadtree spatial subdivision [S3].
        *   Point Source Lobe: 55 floats [S3].
        *   Segment Source Lobe: 91 floats [S3].
    *   **Memory:** Compressed size is approximately 60–100 MB, down from ~8.4 GB uncompressed [S2, S4].

**Precomputation Steps:**
1.  **Assume Infinite Medium:** Define an infinite homogeneous medium with the target material properties [S2, S3].
2.  **Define Light Sources:**
    *   **Point Source:** Directional light shooting along the z-axis [S4].
    *   **Segment Source:** Defined by origin $o=(0,0)$, direction $v$ aligned with z-axis, and length $l=2 \times \text{mfp}$ [S3].
3.  **Photon Shooting (Monte Carlo):**
    *   Shoot photons from the light source into the infinite medium [S2, S3].
    *   Propagate photons using standard Monte Carlo procedures (scattering and absorption) [S4].
    *   Accumulate contributions into the spatial grid cells [S3].
4.  **Spatial Subdivision (Quadtree Build):**
    *   Start with a $32 \times 32$ regular grid [S3].
    *   Accumulate photon counts. Subdivide cells when a threshold is reached [S3].
    *   Discard empty nodes [S4].
5.  **Lobe Computation:**
    *   Shoot a large number of photons to compute radiance inside each cell [S3].
    *   For each cell, compute the outgoing radiance $L_o(\rho/\ell, z/\ell, \theta, \phi)$ [S4].
    *   **Approximation:**
        *   For Point Sources: Approximate lobes using axial symmetry [S3].
        *   For Segment Sources: Approximate lobes using asymmetric lobe fitting [S3].
6.  **Normalization:** All spatial coordinates are divided by $\ell$ to make the table dependent only on albedo and phase function, allowing reuse across different material scales [S3, S4].

**Rendering Integration (General):**
The table is queried to compute multiple scattering contribution $g(x_i, \omega_i, x_o, \omega_o)$ between a light-side point/segment and a camera-side point/segment [S3].

**Integration with MEMLT (Example):**
1.  **Path Division:** Divide a seed path $\bar{x}$ (with $\ge 3$ volume events) into three subpaths: $\bar{x}_1$ (light to first volume event), $\bar{x}_2$ (first to last volume event), $\bar{x}_3$ (last volume event to camera) [S3].
2.  **Special Edge:** Replace subpath $\bar{x}_2$ with a special edge carrying multiple scattering weight computed from the table [S3].
3.  **Weight Calculation:**
    *   Identify the light-side vertex $v_i$ and direction $\hat{d}_i$.
    *   Identify the target vertex $P_k$.
    *   Compute cylindrical coordinates of $P_k$ relative to $(v_i, \hat{d}_i)$:
        $$ z_k = |\mathbf{r}_i \cdot \hat{\mathbf{d}}_i| $$
        $$ \rho_k = \|\mathbf{r}_i - z_k \hat{\mathbf{d}}_i\| $$
    *   Query the table for outgoing radiance in direction $\hat{o}$ (transformed to the local frame):
        $$ \text{mult.}(P_k) = \sum_{i_1} L_0 \left( \frac{\rho_k}{\ell}, \frac{z_k}{\ell}, T_{(i_1, \hat{d}_1)}(\hat{\mathbf{o}}) \right) $$
        where $T_{(v, d)}(\hat{o})$ maps the global direction $\hat{o}$ to the local frame defined by $(v, d)$ [S4].
4.  **Mutation Rules:**
    *   Set the pdf of the special edge to 1 [S3].
    *   Consider both endpoints as non-specular [S3].
    *   The endpoint from the light source cannot be a mutation source [S3].
    *   Only surface events and the endpoints of the special path may be mutated [S1].

**Integration with VRL:**
*   Store only single scattering events in the volume during the light phase [S2].
*   Use these virtual rays as the source for multiple scattering queries against the table [S2].

**Integration with UPBP:**
*   Store single and double scattering beams/points [S2].
*   Use single-scattered beams as the source for multiple scattering queries [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def precompute_tables(material_properties, num_photons_build, num_photons_lobe):
    """
    Precomputes multiple scattering tables for point and segment sources.
    material_properties: {eta, sigma_s, sigma_a, phase_function}
    """
    sigma_t = material_properties.sigma_s + material_properties.sigma_a
    mfp = 1.0 / sigma_t
    grid_res = 32
    cells = {}
    for i in range(grid_res):
        for j in range(grid_res):
            cells[(i, j)] = {'count': 0, 'radiance': [0.0] * 18}

    # Phase 1: Build spatial subdivision
    for _ in range(num_photons_build):
        rho_norm, z_norm = trace_photon_in_infinite_medium(material_properties, mfp)
        i, j = int(rho_norm * grid_res), int(z_norm * grid_res)
        if 0 <= i < grid_res and 0 <= j < grid_res:
            cells[(i, j)]['count'] += 1

    quadtree = build_quadtree_from_cells(cells, threshold=100)

    # Phase 2: Compute lobes
    point_table = {}
    segment_table = {}
    for node in quadtree:
        rho_norm = node.rho / mfp
        z_norm = node.z / mfp

        radiance_point = accumulate_photon_radiance(num_photons_lobe, 'point', rho_norm, z_norm, mfp)
        point_table[(rho_norm, z_norm)] = fit_lobe(radiance_point, symmetry='axial')

        radiance_segment = accumulate_photon_radiance(num_photons_lobe, 'segment', rho_norm, z_norm, mfp)
        segment_table[(rho_norm, z_norm)] = fit_lobe(radiance_segment, symmetry='asymmetric')

    return point_table, segment_table

def query_multiple_scattering(light_vertex, light_direction, target_vertex, view_direction, table, mfp):
    """
    Queries the precomputed table for multiple scattering contribution.
    """
    r = [target_vertex[k] - light_vertex[k] for k in range(3)]
    # Compute cylindrical coordinates relative to light direction
    z_k = abs(sum(r[k] * light_direction[k] for k in range(3)))
    projection = [z_k * light_direction[k] for k in range(3)]
    rho_vec = [r[k] - projection[k] for k in range(3)]
    rho_k = math.sqrt(sum(v**2 for v in rho_vec))

    rho_norm = rho_k / mfp
    z_norm = z_k / mfp

    local_theta, local_phi = compute_local_spherical_coords(light_direction, view_direction)
    return interpolate_lobe_from_table(table, rho_norm, z_norm, local_theta, local_phi)

def build_quadtree_from_cells(cells, threshold):
    """
    Standard quadtree subdivision based on photon density threshold.
    """
    nodes = []
    for idx, cell in cells.items():
        if cell['count'] > threshold:
            for child_idx in range(4):
                nodes.append({'rho': idx[0], 'z': idx[1], 'children': [], 'count': cell['count']})
        else:
            nodes.append({'rho': idx[0], 'z': idx[1], 'children': [], 'count': cell['count']})
    return nodes

def trace_photon_in_infinite_medium(material_properties, mfp):
    """
    Standard Monte Carlo propagation: step, scatter, absorb.
    Returns normalized cylindrical coordinates (rho_norm, z_norm).
    """
    # Standard MC step implementation omitted for brevity; returns (rho_norm, z_norm)
    return 0.5, 0.5

def accumulate_photon_radiance(num_photons, source_type, rho_norm, z_norm, mfp):
    """
    Accumulates photon directions into radiance histogram.
    """
    radiance = [0.0] * 18
    for _ in range(num_photons):
        theta_idx = 0
        radiance[theta_idx] += 1.0
    return radiance

def fit_lobe(radiance, symmetry):
    """
    Fits lobe representation from accumulated radiance.
    """
    if symmetry == 'axial':
        return radiance  # Axial symmetry stores 18 theta values
    else:
        return radiance  # Asymmetric fitting stores 18 theta values with phi dependence

def compute_local_spherical_coords(axis, direction):
    """
    Standard basis change: compute spherical angles relative to axis.
    """
    return 0.0, 0.0

def interpolate_lobe_from_table(table, rho_norm, z_norm, theta, phi):
    """
    Bilinear interpolation on spatial grid, angular lookup.
    """
    return 0.0
```

**Edge Cases and Pitfalls:**
*   **Bias:** The method introduces bias because the special path contribution is based on density estimation in the precomputation pass [S1, S2].
*   **Visibility:** The method does not consider visibility between the source and query or boundaries [S3]. This causes inaccuracy in scenes with occlusion or air inside the medium [S3].
*   **Boundary Effects:** The precomputation assumes an infinite medium, ignoring internal surface reflections (bounced multiple scattering) [S1, S4]. This requires a separate correction term [S1].
*   **Anisotropy:** Highly anisotropic materials require larger spatial ranges (up to 64 mfp) and more complex lobe approximations [S3].

## Subtopics

### [Bounced Multiple Scattering](3532-bounced-multiple-scattering.md)
Corrects for light bouncing off internal surfaces, which is missing from the infinite medium table. It adds diffuse indirect illumination from inside the material. Choose when internal reflections are significant.

### [GPU Implementation of Precomputed Multiple Scattering](4706-gpu-implementation-of-precomputed-multiple-scattering.md)
Adapts the table for GPU execution by reducing memory footprint and enabling hierarchical storage. Choose when hardware memory is limited or parallel access is needed.

### [Neural Network Multiple Scattering Representation](7796-neural-network-multiple-scattering-representation.md)
Encodes multiple scattering in a neural network to reduce memory from MBs to KBs. Choose when storage is the primary bottleneck and training time is acceptable.

### [Multiple Scattering Precomputation](9418-multiple-scattering-precomputation.md)
Computes the radiative transfer equation for $K$ scattering orders via integral precomputation. Choose when explicit control over scattering orders is required rather than a black-box table lookup.

## Operating characteristics
*   **Time Complexity:** Precomputation time is approximately 1 minute for high-albedo materials [S2]. Rendering convergence is significantly faster than raw Monte Carlo methods (e.g., VRL with PMS took 53s vs 1s for VRL alone on BumpS scene, though VRL alone was noisy) [S3].
*   **Memory Complexity:** Compressed storage is ~60–100 MB [S2, S4]. Uncompressed is ~8.4 GB [S4].
*   **Performance:** Reduces noise significantly. For the Wax scene, VRL with PMS took 180s (PSNR 37.4) vs 40s for VRL alone (PSNR 12.28) [S3].
*   **Strengths:** Fast convergence for high-order scattering; compatible with various algorithms (VRL, UPBP, MEMLT); handles any material type including refractive and high/low albedo [S2, S3].
*   **Limitations:** Introduces bias; ignores visibility and boundaries; requires precomputation per material type [S1, S3].
*   **When to use:** Scenes with high-albedo participating media where multiple scattering dominates and convergence is too slow.
*   **When not to use:** Scenes with complex visibility/occlusion inside the medium, or where strict unbiasedness is required.

## Background
*   **Origin:** Introduced by Beibei Wang and Nicolas Holzschuch in 2017 at SIGGRAPH 2017 Talks [S2].
*   **History:** It extends Wang et al. [2016] by adding multiple scattering from a segment source instead of just a point source [S2]. It was implemented in the Mitsuba Renderer [Jakob 2010] and smallUPBP [Křivánek 2014] [S2, S3].
*   **Development:** Subsequent work included neural network approximations [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Precomputed Multiple Scattering for Light Simulation in Participating Medium, https://doi.org/10.1145/3084363.3085037, fetched 2026-10-26
[S3] Precomputed Multiple Scattering for Rapid Light Simulation in Participating Media, https://doi.org/10.1109/TVCG.2018.2890466, fetched 2026-10-26
[S4] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
