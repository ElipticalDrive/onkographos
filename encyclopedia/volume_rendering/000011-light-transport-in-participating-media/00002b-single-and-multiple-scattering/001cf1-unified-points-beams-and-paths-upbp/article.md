# Unified Points, Beams, and Paths (UPBP)
## Overview
Unified Points, Beams, and Paths (UPBP) is a combined rendering algorithm that fuses volumetric radiance estimators (Point-Point, Point-Beam, Beam-Beam) with unbiased bidirectional path tracing (BPT) techniques [S1]. It solves the problem of efficiently simulating light transport in participating media by leveraging the strengths of both biased photon-based methods (for dense, complex scattering) and unbiased path tracing (for long-distance connections) [S1]. Unlike its siblings, UPBP subsumes earlier methods like Volumetric Connection Mapping (VCM) by unifying surface and medium photon mapping with beam-based estimators in a single framework [S1].

## How it works
UPBP operates in two distinct stages: a light subpath tracing stage (Stage 1) and an eye subpath estimation stage (Stage 2) [S1]. The algorithm uses the extended balance heuristic to combine these estimators [S1].

### Inputs and Outputs
*   **Inputs:** Scene geometry, participating media properties (extinction $\sigma_t$, phase function $p$), light sources, camera parameters.
*   **Outputs:** Radiance estimate $L_o$ for each pixel.

### Data Structures
The implementation stores light subpath information in specific acceleration structures for efficient lookup during Stage 2 [S1]:
1.  **Hashed Grids:** Separate hashed grids are built for surface vertices and medium vertices. These store "photon points" and are used for Point-Point (P-P) estimators [S1].
2.  **Bounding Volume Hierarchy (BVH):** A BVH is built over medium vertices. This is used for Point-Beam (P-B) estimators [S1].
3.  **Uniform Grid:** Photon beams (segments of light subpaths passing through media) are organized in a uniform grid. This is used for Beam-Beam (B-B) estimators [S1]. The uniform grid is chosen over a BVH for beams because it is faster to build, although query performance is noted as suboptimal [S1].

### Estimators Used
To balance bias and computational cost, the implementation uses only the **minimum-blur** volumetric estimators [S1]:
*   **P-P3D:** Point-Point 3D estimator for medium vertices.
*   **P-Bl2D:** Point-Beam 2D estimator (using long query beams) for eye subpath segments.
*   **Bs-Bl1D:** Short-Beam Beam-Beam 1D estimator (using short photon beams) for eye subpath segments [S1].

Long-beam variants are excluded because their construction and query costs are high and rarely compensated by variance reduction [S1]. The P-B2D estimator is excluded in favor of P-Bl2D for efficiency [S1].

### Stage 1: Light Subpath Tracing and Storage
1.  Trace a number of light subpaths from light sources.
2.  For each vertex on the light subpath, connect to the eye (light tracing) to generate initial estimates.
3.  Store vertices as "photon points" and segments as "photon beams" [S1].
4.  Build the hashed grids for surface and medium vertices.
5.  Build the BVH for medium vertices.
6.  Build the uniform grid for photon beams.
7.  Store fewer photon beams than points due to higher memory and computational costs per beam [S1].

### Stage 2: Eye Subpath Tracing and Estimation
For each pixel, trace one eye subpath. At each vertex and segment of the eye subpath, evaluate the following estimators [S1]:

1.  **Unbiased BPT Estimators:**
    *   For each vertex (surface or medium), connect to a light source and to vertices of the light subpath to evaluate unbiased BPT estimators [S1].

2.  **Point-Point (P-P) Estimators:**
    *   At each eye subpath vertex, look up photons from the corresponding hashed grid.
    *   Evaluate **P-P2D** if the vertex is on a surface.
    *   Evaluate **P-P3D** if the vertex is in the medium [S1].

3.  **Point-Beam (P-B) and Beam-Beam (B-B) Estimators:**
    *   For each eye subpath segment passing through a medium:
        *   Evaluate **P-Bl2D** (using the BVH for points and long query beams).
        *   Evaluate **Bs-Bl1D** (using the uniform grid for short photon beams) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def unified_points_beams_and_paths(scene, camera, num_light_subpaths):
    """
    Main entry point for the UPBP algorithm.
    """
    # Stage 1: Light Subpath Tracing
    light_subpaths = trace_light_subpaths(scene, num_light_subpaths)
    
    # Store vertices and beams
    surface_vertices = []
    medium_vertices = []
    photon_beams = []
    
    for subpath in light_subpaths:
        for vertex in subpath.vertices:
            if vertex.is_surface:
                surface_vertices.append(vertex)
            else:
                medium_vertices.append(vertex)
        
        for segment in subpath.segments:
            if segment.is_in_medium:
                photon_beams.append(segment)
                
    # Build Acceleration Structures
    surface_grid = build_hashed_grid(surface_vertices)
    medium_grid = build_hashed_grid(medium_vertices)
    medium_bvh = build_bvh(medium_vertices)
    beam_grid = build_uniform_grid(photon_beams)
    
    # Stage 2: Eye Subpath Tracing and Estimation
    image = zeros_like(camera.resolution)
    
    for pixel in camera.pixels:
        eye_subpath = trace_eye_subpath(scene, camera, pixel)
        radiance_estimate = 0.0
        
        for vertex in eye_subpath.vertices:
            # Unbiased BPT connections
            radiance_estimate += evaluate_bpt_estimators(vertex, light_subpaths, scene)
            
            # Point-Point Estimators
            if vertex.is_surface:
                # Look up in surface_grid
                radiance_estimate += evaluate_pp2d(vertex, surface_grid, scene)
            else:
                # Look up in medium_grid
                radiance_estimate += evaluate_pp3d(vertex, medium_grid, scene)
                
        for segment in eye_subpath.segments:
            if segment.is_in_medium:
                # Point-Beam Estimator (Long Query Beam)
                radiance_estimate += evaluate_p_bl2d(segment, medium_bvh, scene)
                
                # Short-Beam Beam-Beam Estimator
                radiance_estimate += evaluate_bs_bl1d(segment, beam_grid, scene)
                
        image[pixel] = radiance_estimate
        
    return image

def trace_light_subpaths(scene, num_subpaths):
    """
    Traces light subpaths from light sources.
    Returns a list of subpaths, each containing vertices and segments.
    """
    subpaths = []
    for _ in range(num_subpaths):
        subpath = []
        current_pos = sample_light_source(scene)
        current_dir = sample_light_direction(scene, current_pos)
        
        # Trace path... (Implementation details omitted for brevity, assumes standard path tracing)
        # Vertices and segments are collected.
        pass 
    return subpaths

def trace_eye_subpath(scene, camera, pixel):
    """
    Traces one eye subpath for a given pixel.
    Returns an eye subpath object with vertices and segments.
    """
    # Implementation details omitted for brevity
    pass

def evaluate_bpt_estimators(vertex, light_subpaths, scene):
    """
    Evaluates unbiased BPT estimators by connecting vertex to light sources
    and light subpath vertices.
    """
    # Implementation details omitted for brevity
    pass

def evaluate_pp2d(vertex, grid, scene):
    """
    Evaluates P-P2D estimator for surface vertices.
    Looks up photons in the surface hashed grid.
    """
    # Implementation details omitted for brevity
    pass

def evaluate_pp3d(vertex, grid, scene):
    """
    Evaluates P-P3D estimator for medium vertices.
    Looks up photons in the medium hashed grid.
    """
    # Implementation details omitted for brevity
    pass

def evaluate_p_bl2d(segment, bvh, scene):
    """
    Evaluates P-Bl2D estimator for medium segments.
    Uses long query beams and medium BVH.
    """
    # Implementation details omitted for brevity
    pass

def evaluate_bs_bl1d(segment, grid, scene):
    """
    Evaluates Bs-Bl1D estimator for medium segments.
    Uses short photon beams and uniform grid.
    """
    # Implementation details omitted for brevity
    pass
```

## Operating characteristics
*   **Complexity:** The construction of acceleration structures (hashed grids, BVH, uniform grid) depends on the number of light subpaths and vertices/beams. Query performance for the uniform grid (beams) is noted as suboptimal compared to BVH, but the uniform grid is faster to build [S1].
*   **Performance:** The use of short photon beams and long query beams (P-P3D, P-Bl2D, Bs-Bl1D) provides the best performance for a given time budget [S1]. Long photon beams are avoided due to high construction and query costs [S1].
*   **Strengths:** Subsumes previous methods (VCM, photon mapping) and handles both surface and medium transport efficiently. BPT components excel at connecting subpaths over long distances [S1].
*   **Limitations:** Does not use joint path importance sampling techniques, which could further reduce variance in scenes with highly anisotropically scattering media [S1]. The uniform grid for beams has suboptimal query performance [S1].

## Background
UPBP was introduced by Jarosz et al. in 2011/2012 [S1]. It builds upon the Volumetric Connection Mapping (VCM) algorithm by Georgiev et al. [2012] and photon beam methods by Jarosz et al. [2011b] [S1]. The implementation builds on the publicly available SmallVCM framework [S1].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
