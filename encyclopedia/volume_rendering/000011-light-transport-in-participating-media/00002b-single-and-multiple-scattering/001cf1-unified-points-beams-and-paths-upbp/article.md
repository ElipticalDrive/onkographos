# Unified Points, Beams, and Paths (UPBP)


## Overview
Unified Points, Beams, and Paths (UPBP) is a combined rendering algorithm that fuses volumetric density estimation estimators (Point-Point, Point-Beam, Beam-Beam) with unbiased Bidirectional Path Tracing (BPT) using Multiple Importance Sampling (MIS) [S2, S3]. It solves the problem of efficiently simulating light transport in diverse participating media by automatically selecting the most suitable representation—points for high-order scattering, beams for low-order scattering, and paths for specular surface interactions—thereby mitigating the weaknesses of individual estimators [S1, S3]. Unlike pure path tracing, which struggles with low-order scattering in dense media, or pure photon mapping, which suffers from bias and poor handling of specular connections, UPBP provides a unified framework that subsumes previous methods like Volumetric Connection Mapping (VCM) and Photon Beam Tracing [S2].

## How it works
The algorithm operates in two distinct stages: a light subpath generation phase (Stage 1) and a camera subpath evaluation phase (Stage 2) [S2, S3]. It uses the extended balance heuristic for MIS weights to combine contributions from different estimators [S3].

### Inputs and Outputs
*   **Inputs:** Scene geometry, participating media properties (extinction $\sigma_t$, scattering $\sigma_s$, emission $\sigma_e$, phase function $p$), light sources, camera parameters, and user-defined parameters: maximum path length $k$, number of light subpaths $n_v$ for each technique $v$, and the set of active estimators.
*   **Outputs:** Radiance estimate for each pixel.

### Data Structures
The implementation maintains specific data structures to accelerate density estimation queries [S2, S3]:
1.  **Hashed Grids:** Separate grids for surface vertices and medium vertices. Used for Point-Point (P-P) estimators.
2.  **Bounding Volume Hierarchy (BVH):** Built over medium vertices. Used for Point-Beam (P-B) estimators.
3.  **Uniform Grid:** Organizes photon beams. Used for Beam-Beam (B-B) estimators. The uniform grid is chosen over BVH for beams because it is faster to build, though query performance is suboptimal [S2].

### Active Estimators
To reduce bias and computational cost, the standard implementation uses only the "minimum-blur" volumetric estimators and their short-beam variants [S2]:
*   **P-P3D:** Point-Point 3D (medium).
*   **P-Bl2D:** Point-Beam 2D (short photon beams, long query beams).
*   **Bs-Bl1D:** Beam-Beam 1D (short beams).
*   **P-P2D:** Point-Point 2D (surface).
*   **BPT:** Bidirectional Path Tracing.

Long photon beams are excluded due to high construction and query costs that are rarely compensated by variance reduction [S2].

### Numbering and Indices
*   $k$: Maximum path length (number of bounces).
*   $s$: Number of vertices on the light subpath.
*   $t$: Number of vertices on the camera subpath.
*   For BPT: $t = k + 1 - s$.
*   For PDE (Point/Beam Density Estimation): $t = k + 2 - s$.
*   PDE techniques cannot be evaluated on a light source or camera lens, so valid indices are $2 \leq s \leq k$ [S3].

### Step-by-Step Algorithm

**Stage 1: Light Subpath Generation and Storage**
1.  Generate $n_{BPT}$ light subpaths $\bar{\bf y}$ using standard path tracing or light tracing techniques.
2.  Store vertices on surfaces and vertices/beams in media.
3.  Build the hashed grids for surface and medium vertices.
4.  Build the BVH over medium vertices.
5.  Organize photon beams into the uniform grid.
6.  Store fewer photon beams than points due to higher memory and computational costs per beam [S2].

**Stage 2: Camera Subpath Evaluation**
1.  For each pixel, generate one camera subpath $\bar{\bf z}$ [S3].
2.  Stochastically terminate light and camera subpaths up to the maximum path length $k$ [S3].
3.  For each vertex on the camera subpath (surface or medium):
    *   **BPT Evaluation:** Connect the camera vertex to all vertices of a randomly chosen stored light subpath to evaluate the BPT estimator [S2, S3].
    *   **P-P Evaluation:**
        *   If the vertex is on a surface, evaluate the P-P2D estimator by looking up photons from the surface hashed grid [S2].
        *   If the vertex is in the medium, evaluate the P-P3D estimator by looking up photons from the medium hashed grid [S2].
4.  For each segment (beam) on the camera subpath passing through a medium:
    *   Evaluate the P-Bl2D estimator using the medium BVH and the uniform grid for beams [S2].
    *   Evaluate the Bs-Bl1D estimator using the uniform grid for beams [S2].
5.  For each evaluated estimator $v$ at vertex/segment $s$, compute the MIS weight $w_{v,s,t}$ and the radiance contribution $\langle I \rangle_{v,s}$.
6.  Sum the weighted contributions and add to the pixel value.

### Formulas

The total radiance estimate $\langle I \rangle_{UPBP}$ is the sum of contributions from BPT and the four PDE techniques [S3]:

$$
\langle I \rangle_{UPBP} = \frac{1}{n_{BPT}} \sum_{i=1}^{n_{BPT}} \sum_{s=0}^{k+1} w_{BPT,s,t}(\bar{\bf x}_i) \langle I \rangle_{BPT,s}(\bar{\bf x}_i) + \frac{1}{n_{B-B1D}} \sum_{i=1}^{n_{B-B1D}} \sum_{s=2}^k w_{B-B1D,s,t}(\bar{\bf x}_i) \langle I \rangle_{B-B1D,s}(\bar{\bf x}_i) + \frac{1}{n_{P-B2D}} \sum_{i=1}^{n_{P-B2D}} \sum_{s=2}^k w_{P-B2D,s,t}(\bar{\bf x}_i) \langle I \rangle_{P-B2D,s}(\bar{\bf x}_i) + \frac{1}{n_{P-P3D}} \sum_{i=1}^{n_{P-P3D}} \sum_{s=2}^k w_{P-P3D,s,t}(\bar{\bf x}_i) \langle I \rangle_{P-P3D,s}(\bar{\bf x}_i) + \frac{1}{n_{P-P2D}} \sum_{i=1}^{n_{P-P2D}} \sum_{s=2}^k w_{P-P2D,s,t}(\bar{\bf x}_i) \langle I \rangle_{P-P2D,s}(\bar{\bf x}_i)
$$

Where $n_v$ is the number of light subpaths for technique $v$, and $w_{v,s,t}$ is the MIS weight [S3].

The MIS weight is computed using the extended balance heuristic [S3]:

$$
w_{v,s,t}(\bar{\bf x}) = \frac{n_v p_{v,s}(\bar{\bf x})}{g_v(\bar{\bf x})}
$$

$$
g_v(\bar{\bf x}) = \sum_{j=0}^{k+1} n_{BPT} p_{BPT,j}(\bar{\bf x}) + \sum_{j=2}^k n_{B-B1D} p_{B-B1D,j}(\bar{\bf x}) + n_{P-B2D} p_{P-B2D,j}(\bar{\bf x}) + \sum_{j=2}^k n_{P-P3D} p_{P-P3D,j}(\bar{\bf x}) + n_{P-P2D} p_{P-P2D,j}(\bar{\bf x})
$$

Here, $p_{v,s}(\bar{\bf x})$ denotes the probability density function (PDF) of a path created by technique $v$ from a subpath with $s$ vertices [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upbp_render(scene, camera, max_path_length_k, n_light_subpaths):
    """
    Main UPBP rendering loop.
    """
    # Stage 1: Light Subpath Generation
    light_subpaths = []
    surface_vertices = []
    medium_vertices = []
    photon_beams = []
    
    for i in range(n_light_subpaths):
        # Generate a light subpath y_bar
        y_bar = trace_light_subpath(scene, max_path_length_k)
        light_subpaths.append(y_bar)
        
        # Store vertices and beams
        for vertex in y_bar.vertices:
            if vertex.is_surface:
                surface_vertices.append(vertex)
            else:
                medium_vertices.append(vertex)
        
        # Store beams (segments between medium vertices)
        for beam in y_bar.beams:
            photon_beams.append(beam)

    # Build Data Structures
    surface_grid = build_hashed_grid(surface_vertices)
    medium_grid = build_hashed_grid(medium_vertices)
    medium_bvh = build_bvh(medium_vertices)
    beam_grid = build_uniform_grid(photon_beams)

    # Stage 2: Camera Subpath Evaluation
    image = zeros(camera.resolution)
    
    for pixel in camera.pixels:
        # Generate one camera subpath per pixel
        z_bar = trace_camera_subpath(camera, max_path_length_k)
        
        pixel_radiance = 0.0
        
        # Evaluate BPT and P-P estimators at vertices
        for s, vertex in enumerate(z_bar.vertices):
            # BPT: Connect to a random light subpath
            # Note: In practice, BPT connects to ALL light subpath vertices, 
            # but the formula averages over n_BPT. 
            # The source [S3] says "evaluated with all vertices of a randomly chosen light subpath"
            # but the formula sums over n_BPT. We follow the formula structure.
            
            # Calculate MIS weight denominator g_v(x)
            # This requires PDFs for all techniques. 
            # For implementation, we compute contributions and weights.
            
            # BPT Contribution
            bpt_contrib = 0.0
            for light_subpath in light_subpaths:
                # Connect vertex to light subpath vertices
                # Evaluate BPT estimator <I>_BPT,s
                bpt_contrib += evaluate_bpt_estimator(vertex, light_subpath)
            bpt_contrib /= n_light_subpaths
            
            # P-P2D (Surface) or P-P3D (Medium) Contribution
            pp_contrib = 0.0
            if vertex.is_surface:
                # Query surface_grid
                pp_contrib = evaluate_pp2d_estimator(vertex, surface_grid)
            else:
                # Query medium_grid
                pp_contrib = evaluate_pp3d_estimator(vertex, medium_grid)
                
            # Compute MIS weights and sum
            # w_v,s,t = n_v * p_v,s / g_v
            # Contribution = w * <I>
            # Note: The formula divides by n_v outside the sum. 
            # The inner sum is over s.
            # We accumulate weighted contributions.
            
            # Simplified accumulation logic based on the formula structure:
            # The formula sums over i (samples) and s (vertices).
            # Here we have 1 camera sample (z_bar).
            
            # We need to compute the weight for each technique at this vertex/segment.
            # This requires the PDFs.
            
            # For BPT:
            w_bpt = compute_mis_weight('BPT', s, z_bar, light_subpaths, surface_grid, medium_grid, beam_grid, medium_bvh)
            pixel_radiance += w_bpt * bpt_contrib
            
            # For P-P:
            w_pp = compute_mis_weight('PP', s, z_bar, light_subpaths, surface_grid, medium_grid, beam_grid, medium_bvh)
            pixel_radiance += w_pp * pp_contrib

        # Evaluate P-B and B-B estimators at segments (beams)
        for s, beam in enumerate(z_bar.beams):
            # P-Bl2D (Point-Beam)
            p_bl2d_contrib = evaluate_p_bl2d_estimator(beam, medium_bvh, beam_grid)
            
            # Bs-Bl1D (Beam-Beam)
            bs_bl1d_contrib = evaluate_bs_bl1d_estimator(beam, beam_grid)
            
            # Compute MIS weights for beam estimators
            w_p_bl2d = compute_mis_weight('P-B', s, z_bar, light_subpaths, surface_grid, medium_grid, beam_grid, medium_bvh)
            w_bs_bl1d = compute_mis_weight('B-B', s, z_bar, light_subpaths, surface_grid, medium_grid, beam_grid, medium_bvh)
            
            pixel_radiance += w_p_bl2d * p_bl2d_contrib
            pixel_radiance += w_bs_bl1d * bs_bl1d_contrib
            
        image[pixel] = pixel_radiance

    return image

def compute_mis_weight(technique, s, z_bar, light_subpaths, surface_grid, medium_grid, beam_grid, medium_bvh):
    """
    Computes the MIS weight w_v,s,t using the extended balance heuristic.
    """
    # Numerator: n_v * p_v,s
    n_v = get_n_light_subpaths(technique)
    p_v_s = get_pdf(technique, s, z_bar, light_subpaths, surface_grid, medium_grid, beam_grid, medium_bvh)
    numerator = n_v * p_v_s
    
    # Denominator: sum over all techniques j of n_j * p_j,s
    denominator = 0.0
    for technique_j in ['BPT', 'B-B1D', 'P-B2D', 'P-P3D', 'P-P2D']:
        n_j = get_n_light_subpaths(technique_j)
        p_j_s = get_pdf(technique_j, s, z_bar, light_subpaths, surface_grid, medium_grid, beam_grid, medium_bvh)
        denominator += n_j * p_j_s
        
    return numerator / denominator
```

### Edge Cases and Pitfalls
*   **Bias vs. Variance:** Using long photon beams increases construction/query costs without significant variance reduction. The implementation opts for short photon beams [S2].
*   **Estimator Selection:** Not all 25 variations of volumetric estimators are combined. Similar PDFs (e.g., P-B2D and P-B3D) are not both used to avoid redundancy; only minimum-blur estimators are selected [S2].
*   **PDE Limitations:** PDE techniques cannot be evaluated on light sources or camera lenses, requiring subpaths to have at least two vertices ($2 \leq s \leq k$) [S3].
*   **Query Performance:** The uniform grid for beams is faster to build than a BVH but has suboptimal query performance [S2].

## Subtopics

### [GPU-accelerated UPBP with RTX BVH](10615-gpu-accelerated-upbp-with-rtx-bvh.md)
A full GPU implementation leveraging NVIDIA RTX hardware for light and camera path generation, using BVH for accelerated data structures and a constant-time MIS weight algorithm. It solves the CPU performance bottleneck by adapting Point-Point and Beam-Beam estimators to ray-tracing acceleration structures.

## Operating characteristics
*   **Time Complexity:** The algorithm involves two stages. Stage 1 scales with the number of light subpaths and the cost of building data structures (hashing, BVH, uniform grid). Stage 2 scales with the number of pixels and the cost of querying data structures for each vertex and segment of the camera subpath. The uniform grid for beams is noted to have suboptimal query performance [S2].
*   **Memory Complexity:** Memory usage is dominated by the storage of light subpaths, vertices, and beams. The implementation stores fewer photon beams than points due to higher memory costs per beam [S2].
*   **Performance:** The use of short photon beams and long query beams provides the best performance for a given time budget [S2]. The GPU-accelerated variant achieves significant speedups over CPU baselines [S1].
*   **Strengths:** Automatically selects the best representation for different media types (points for high-order, beams for low-order, paths for specular) [S1]. Subsumes previous methods like VCM and Photon Beam Tracing [S2].
*   **Limitations:** The uniform grid for beams has suboptimal query performance [S2]. The algorithm does not use joint path importance sampling techniques, which could further reduce variance in scenes with highly anisotropically scattering media [S2].

## Background
UPBP was introduced by Křivánek et al. [S1, S2, S3]. It builds on the observation that different representations (points, beams, paths) are suitable for different types of participating media [S1]. The method combines Bidirectional Path Tracing (BPT) with Point/Beam Density Estimation (PDE) estimators using Multiple Importance Sampling (MIS) [S3]. It subsumes the methods of Georgiev et al. [2012] (Volumetric Connection Mapping) and Hachisuka et al. [2012] (Photon Beam Tracing) [S2].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
[S3] Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation, https://doi.org/10.1111/cgf.14802, fetched 2026-10-27
