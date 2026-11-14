# Unified Points, Beams, and Paths (UPBP)


## Overview
Unified Points, Beams, and Paths (UPBP) is a combined rendering algorithm that fuses volumetric density estimation estimators (Point-Point, Point-Beam, Beam-Beam) with unbiased bidirectional path tracing (BPT) using Multiple Importance Sampling (MIS) [S2, S3]. It solves the problem of efficiently simulating light transport in diverse participating media by automatically selecting the most suitable representation—points for high-order scattering, beams for low-order scattering, and paths for volume effects with specular surfaces—thereby mitigating the weaknesses of individual estimators [S1, S3]. Unlike its siblings, UPBP subsumes methods like [VCM](873-path-integral-framework.md) and [Photon Mapping](985-delta-tracking.md) by treating them as special cases within a unified MIS framework [S2].

## How it works
UPBP operates in two distinct stages per iteration: a light subpath generation stage and a camera subpath evaluation stage [S2, S3].

### Inputs and Outputs
*   **Inputs:** Scene geometry, participating media properties (extinction $\sigma_t$, scattering $\sigma_s$, phase function $p$), light sources, camera parameters, maximum path length $k$, and number of light subpaths $n_v$ for each technique $v$ [S2, S3].
*   **Outputs:** Radiance estimate for each pixel [S2, S3].

### Data Structures
The algorithm maintains specific data structures to accelerate density estimation queries during the camera stage [S2]:
1.  **Hashed Grids:** Separate hashed grids for surface vertices and medium vertices to support Point-Point (P-P) estimators [S2].
2.  **Bounding Volume Hierarchy (BVH):** Built over medium vertices to support Point-Beam (P-B) estimators [S2].
3.  **Uniform Grid:** Organizes photon beams to support Beam-Beam (B-B) estimators. This was found faster than a BVH for beam queries, though query performance is suboptimal [S2].

### Estimators Used
The implementation uses a subset of estimators to balance bias and computational cost [S2]:
*   **P-P3D:** Point-Point 3D for medium vertices.
*   **P-B2D:** Point-Beam 2D for medium vertices (specifically using short photon beams and long query beams, denoted P-Bl2D in some contexts, but formally P-B2D in the estimator family) [S2].
*   **B-B1D:** Beam-Beam 1D (specifically short-beam variant Bs-Bl1D) for medium segments [S2].
*   **P-P2D:** Point-Point 2D for surface vertices [S2].
*   **BPT:** Bidirectional Path Tracing for unbiased connections [S2, S3].

Long-beam variants and P-B3D are excluded due to high construction/query costs or similar PDFs to other estimators [S2].

### Numbered Steps

**Stage 1: Light Subpath Generation and Storage**
1.  Generate $n_{BPT}$ light subpaths $\bar{\bf y}$ [S3].
2.  For each light subpath, store vertices on surfaces and vertices/beams in media [S3].
3.  Build the hashed grids over surface and medium vertices [S2].
4.  Build the BVH over medium vertices [S2].
5.  Organize photon beams in a uniform grid [S2].

**Stage 2: Camera Subpath Evaluation**
1.  For each pixel, generate one camera subpath $\bar{\bf z}$ [S3].
2.  Stochastically terminate light and camera subpaths up to the user-defined maximum path length $k$ [S3].
3.  For each vertex $s$ on the camera subpath (where $0 \le s \le k+1$ for BPT, $2 \le s \le k$ for PDE):
    *   **BPT Evaluation:** Connect the camera vertex to all vertices of a randomly chosen stored light subpath to evaluate the BPT estimator $\langle I \rangle_{BPT,s}$ [S2, S3].
    *   **PDE Evaluation:** Query the data structures for nearby beams/points to evaluate PDE estimators:
        *   If the vertex is on a surface, evaluate P-P2D $\langle I \rangle_{P-P2D,s}$ [S2].
        *   If the vertex is in the medium, evaluate P-P3D $\langle I \rangle_{P-P3D,s}$ [S2].
4.  For each segment (beam) on the camera subpath passing through a medium:
    *   Evaluate P-B2D $\langle I \rangle_{P-B2D,s}$ and B-B1D $\langle I \rangle_{B-B1D,s}$ estimators by querying the BVH and uniform grid respectively [S2].
5.  Compute the MIS weight $w_{v,s,t}(\bar{\bf x})$ for each technique $v$ and path configuration $(s,t)$ [S3].
6.  Accumulate the weighted contributions to the pixel value [S3].

### Formulas
The total radiance estimate is the sum of contributions from all techniques $v \in \{BPT, B-B1D, P-B2D, P-P3D, P-P2D\}$ [S3]:

$$
\langle I \rangle_{UPBP} = \sum_{v} \frac{1}{n_v} \sum_{i=1}^{n_v} \sum_{s} w_{v,s,t}(\bar{\bf x}_i) \langle I \rangle_{v,s}(\bar{\bf x}_i)
$$

Where:
*   $n_v$ is the number of light subpaths for technique $v$.
*   $s$ is the number of vertices on the light subpath.
*   $t$ is the number of vertices on the camera subpath.
    *   For BPT: $t = k + 1 - s$ [S3].
    *   For PDE (volumetric estimators): $t = k + 2 - s$ [S3]. Note that PDE techniques require at least two vertices ($2 \le s \le k$) as they cannot be evaluated on light sources or camera lenses [S3].
*   $\langle I \rangle_{v,s}$ is the radiance estimate from technique $v$ at light subpath vertex $s$ [S3].

The MIS weight uses the extended balance heuristic [S3]:

$$
w_{v,s,t}(\bar{\bf x}) = \frac{n_v p_{v,s}(\bar{\bf x})}{g_v(\bar{\bf x})}
$$

$$
g_v(\bar{\bf x}) = \sum_{j=0}^{k+1} n_{BPT} p_{BPT,j}(\bar{\bf x}) + \sum_{j=2}^k n_{B-B1D} p_{B-B1D,j}(\bar{\bf x}) + n_{P-B2D} p_{P-B2D,j}(\bar{\bf x}) + \sum_{j=2}^k n_{P-P3D} p_{P-P3D,j}(\bar{\bf x}) + n_{P-P2D} p_{P-P2D,j}(\bar{\bf x})
$$

Where $p_{v,s}(\bar{\bf x})$ is the probability density function of generating the path configuration with technique $v$ and light subpath length $s$ [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upbp_render(scene, camera, max_path_length_k, n_light_subpaths):
    """
    Main UPBP rendering loop.
    """
    # Initialize data structures
    surface_grid = HashedGrid()
    medium_grid = HashedGrid()
    medium_bvh = BVH()
    beam_grid = UniformGrid()
    
    # Stage 1: Light Subpath Generation
    light_subpaths = []
    for _ in range(n_light_subpaths):
        # Generate light subpath y_bar
        y_bar = generate_light_subpath(scene, max_path_length_k)
        
        # Store vertices and beams
        for vertex in y_bar.vertices:
            if vertex.is_surface:
                surface_grid.insert(vertex)
            else:
                medium_grid.insert(vertex)
                medium_bvh.insert(vertex)
        
        for beam in y_bar.beams:
            beam_grid.insert(beam)
            
        light_subpaths.append(y_bar)
        
    # Stage 2: Camera Subpath Evaluation
    for pixel in camera.raster:
        pixel_value = 0.0
        
        # Generate one camera subpath per pixel
        z_bar = generate_camera_subpath(camera, max_path_length_k)
        
        # Evaluate BPT and PDE estimators at vertices
        for s, vertex in enumerate(z_bar.vertices):
            # Determine t based on technique type
            # BPT: t = k + 1 - s
            # PDE: t = k + 2 - s
            
            # BPT Estimator
            # Connect to a randomly chosen light subpath
            y_bar = random.choice(light_subpaths)
            bpt_contrib = evaluate_bpt(vertex, y_bar)
            bpt_weight = compute_mis_weight('BPT', s, len(z_bar.vertices) - 1 - s, bpt_contrib.pdf)
            pixel_value += bpt_weight * bpt_contrib
            
            # PDE Estimators (if not on light/camera lens)
            if 2 <= s <= max_path_length_k:
                # P-P2D for surface
                if vertex.is_surface:
                    pp2d_contrib = evaluate_pp2d(vertex, surface_grid)
                    pp2d_weight = compute_mis_weight('P-P2D', s, len(z_bar.vertices) - 2 - s, pp2d_contrib.pdf)
                    pixel_value += pp2d_weight * pp2d_contrib
                    
                # P-P3D for medium
                else:
                    pp3d_contrib = evaluate_pp3d(vertex, medium_grid)
                    pp3d_weight = compute_mis_weight('P-P3D', s, len(z_bar.vertices) - 2 - s, pp3d_contrib.pdf)
                    pixel_value += pp3d_weight * pp3d_contrib
                    
                # P-B2D for medium (using beam from vertex)
                # Note: P-B2D is evaluated on segments in the source, 
                # but the formula sums over s. The source says "At every camera beam... evaluate PDE".
                # We handle beams in the next loop.
        
        # Evaluate PDE estimators on beams (segments)
        for s, beam in enumerate(z_bar.beams):
            # Beam connects vertex s to s+1
            # P-B2D: Point (from light) to Beam (camera)
            pb2d_contrib = evaluate_pb2d(beam, medium_grid, medium_bvh)
            pb2d_weight = compute_mis_weight('P-B2D', s, len(z_bar.vertices) - 2 - s, pb2d_contrib.pdf)
            pixel_value += pb2d_weight * pb2d_contrib
            
            # B-B1D: Beam (light) to Beam (camera)
            bb1d_contrib = evaluate_bb1d(beam, beam_grid)
            bb1d_weight = compute_mis_weight('B-B1D', s, len(z_bar.vertices) - 2 - s, bb1d_contrib.pdf)
            pixel_value += bb1d_weight * bb1d_contrib
            
        pixel.value = pixel_value

def compute_mis_weight(technique, s, t, pdf_v):
    """
    Computes the MIS weight using the extended balance heuristic.
    """
    # Calculate denominator g_v(x)
    # Sum over all techniques and valid s ranges
    denominator = 0.0
    
    # BPT: s from 0 to k+1
    for j in range(0, max_path_length_k + 2):
        # PDF for BPT at s=j
        pdf_bpt_j = get_pdf_bpt(j) 
        denominator += n_light_subpaths['BPT'] * pdf_bpt_j
        
    # B-B1D: s from 2 to k
    for j in range(2, max_path_length_k + 1):
        pdf_bb1d_j = get_pdf_bb1d(j)
        denominator += n_light_subpaths['B-B1D'] * pdf_bb1d_j
        
    # P-B2D: s from 2 to k (single term in sum in source, but implies range)
    # Source: n_{P-B2D} p_{P-B2D,j}
    # Assuming j ranges similarly to other PDE techniques
    for j in range(2, max_path_length_k + 1):
        pdf_pb2d_j = get_pdf_pb2d(j)
        denominator += n_light_subpaths['P-B2D'] * pdf_pb2d_j
        
    # P-P3D: s from 2 to k
    for j in range(2, max_path_length_k + 1):
        pdf_pp3d_j = get_pdf_pp3d(j)
        denominator += n_light_subpaths['P-P3D'] * pdf_pp3d_j
        
    # P-P2D: s from 2 to k
    for j in range(2, max_path_length_k + 1):
        pdf_pp2d_j = get_pdf_pp2d(j)
        denominator += n_light_subpaths['P-P2D'] * pdf_pp2d_j
        
    # Numerator
    numerator = n_light_subpaths[technique] * pdf_v
    
    return numerator / denominator if denominator > 0 else 0.0
```

### Parameters
*   **Maximum Path Length ($k$):** User-defined limit for stochastic termination of light and camera subpaths [S3].
*   **Number of Light Subpaths ($n_v$):** Number of light subpaths generated for each technique $v$ [S3].
*   **Representation Choice:** Photons for high-order scattering, beams for low-order scattering, paths for volume effects with specular surfaces [S1].

### Edge Cases and Pitfalls
*   **PDE Evaluation Constraints:** PDE techniques cannot be evaluated on a light source or camera lens, requiring subpaths to have at least two vertices ($2 \le s \le k$) [S3].
*   **Beam Query Performance:** The uniform grid for photon beams is faster to build than a BVH but has suboptimal query performance [S2].
*   **Long Beams:** Long photon beams have high construction and query costs, which are rarely compensated by variance reduction; thus, short photon beams are preferred [S2].

## Subtopics

### [GPU-accelerated UPBP with RTX BVH](10615-gpu-accelerated-upbp-with-rtx-bvh.md)
A full GPU implementation leveraging NVIDIA RTX hardware for path generation and accelerated data structures (BVH) for density estimation, achieving significant speedups over CPU baselines by adapting Point-Point and Beam-Beam estimators to ray-tracing acceleration structures.

## Operating characteristics
*   **Time Complexity:** Dominated by the generation of light subpaths and the construction of acceleration structures (hashed grids, BVH, uniform grid) in Stage 1, and the evaluation of multiple estimators per camera vertex/segment in Stage 2 [S2].
*   **Memory Complexity:** Proportional to the number of stored light subpath vertices and beams, as well as the size of the acceleration structures [S2].
*   **Strengths:** Automatically selects the best estimator for the scene (points for high-order, beams for low-order, paths for specular volumes), subsuming previous methods like VCM and Photon Mapping [S1, S2].
*   **Limitations:** High computational cost per pixel due to the evaluation of multiple estimators and the overhead of managing multiple data structures [S2]. Query performance for beam-based estimators can be suboptimal [S2].
*   **When to Use:** Scenes with diverse participating media where no single estimator (pure BPT, pure Photon Mapping, or pure Beam Tracing) is efficient across the entire scene [S1].
*   **When Not to Use:** Simple scenes where a single estimator suffices, or when memory is severely constrained due to the storage of multiple data structures [S2].

## Background
UPBP was introduced by Křivánek et al. in 2014 [S1, S2, S3]. It builds on the observation that different representations (points, beams, paths) are suitable for different types of light transport in participating media [S1]. The method unifies Bidirectional Path Tracing (BPT) with Volumetric Density Estimation (VDE) estimators (P-P, P-B, B-B) using Multiple Importance Sampling (MIS) [S2, S3]. It subsumes earlier methods such as Vertex Connection and Merging (VCM) [S2] and Photon Mapping for participating media [S2].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
[S3] Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation, https://doi.org/10.1111/cgf.14802, fetched 2026-10-27
