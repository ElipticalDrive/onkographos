# Sorting-free Pre-integrated Projected Tetrahedra

Also called: Pre-integrated Projected Tetrahedra, Sorting-free approach that controls the fragment rendering by using the opacity value, Sorting-free technique for volume rendering of an irregular volume dataset

## Overview
Sorting-free Pre-integrated Projected Tetrahedra (PBVR) is a stochastic volume rendering technique for irregular volume datasets that eliminates the need for explicit back-to-front visibility sorting [S1]. Unlike conventional Projected Tetrahedra which requires sorting cells to correctly composite semi-transparent fragments, PBVR treats rendering as a stochastic process where fragments are rendered with a probability equal to their opacity, achieving correct compositing through multiple repetitions [S1]. It solves the scalability issues of sorting-based methods on large-scale irregular volumes and the visible footprint artifacts of particle-based methods [S1].

## How it works
**Inputs:**
*   Irregular volume dataset decomposed into tetrahedral cells.
*   Scalar field values at tetrahedron vertices.
*   Transfer function mapping scalar values to density ($\rho$) and luminosity ($c$).
*   Camera parameters (viewing ray definition).
*   Repetition level $N$ (number of stochastic passes).

**Outputs:**
*   Final image brightness/color per pixel, computed as the ensemble average over $N$ repetitions.

**Data Structures:**
*   **Tetrahedron:** Defined by 4 vertices in 3D space.
*   **Projected Tetrahedron:** 2D projection of the tetrahedron onto the image plane, subdivided into polygons corresponding to front/back faces [S1].
*   **Fragment:** Rasterized pixel coverage within a projected tetrahedron polygon.
*   **Ray Segment:** A portion of the viewing ray intersecting a tetrahedron, characterized by constant density $\rho_k$ and luminosity $c_k$ [S1].

**Mathematical Model:**
The method uses a "density emitter model" where the scalar field is characterized as a varying density emitter with single-level scattering [S1]. The number of particles in a volume space is assumed to follow a Poisson distribution [S1].

1.  **Opacity Calculation:**
    For a ray segment of length $\Delta t$ with particle density $\rho$ (particles per unit volume), the opacity $\tau$ is derived from the Poisson distribution assumption [S1]:
    $$ \tau = 1 - e^{-\rho \cdot \pi r^2 \cdot \Delta t} $$
    Where $\pi r^2 \rho \Delta t$ expresses the expected number of particles in the ray segment [S1]. Note: In the context of projected tetrahedra, $\Delta t$ corresponds to the depth extent of the intersection, and the cross-sectional area factor is implicitly handled by the projection and rasterization process. The source defines opacity generally as $1 - e^{-\rho \cdot \text{area} \cdot \text{length}}$ [S1].

2.  **Brightness Calculation (Deterministic Reference):**
    The brightness $B_k$ for a segment $k$ with constant luminosity $c_k$ and opacity $\tau_k$ is:
    $$ B_k = c_k (1 - e^{-\rho \cdot \text{area} \cdot \Delta t}) $$
    Or more generally, if intermediate brightness $B_k$ is considered in a back-to-front accumulation, the complexity is $O(n^2)$ [S1].

3.  **Stochastic Rendering (Sorting-Free Approach):**
    Instead of sorting, the method repeats the rendering process $N$ times. In each repetition, fragments are rendered based on a probabilistic discard rule [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize the accumulated image buffer to zero. Set repetition counter $n = 0$.
2.  **Loop:** While $n < N$:
    a.  **Projection:** Project all tetrahedral cells to 2D screen space. Subdivide the projected area into polygons based on front/back face intersections [S1].
    b.  **Rasterization:** For each polygon, generate fragments.
    c.  **Opacity Evaluation:** For each fragment, evaluate the scalar value at the fragment's position (using linear interpolation within the tetrahedron) and map it to density $\rho$ and luminosity $c$ via the transfer function [S1]. Calculate the segment opacity $\tau$ [S1].
    d.  **Stochastic Discard:** Generate a random number $r \in [0, 1]$.
        *   If $r > \tau$, discard the fragment (do not write to the buffer).
        *   If $r \le \tau$, render the fragment with its calculated luminosity/color [S1].
        *   *Note:* The source states: "The possibility that the k-th luminosity is equal to the brightness value is defined by... This represents an event where there is no particle from the first to the (k-1)-th ray-segments and more than one particle in the k-th ray segment." The practical implementation described is controlling fragment rendering by opacity probability [S1].
    e.  **Accumulation:** Add the rendered fragments to the accumulated image buffer.
    f.  Increment $n$.
3.  **Finalization:** Divide the accumulated image buffer by $N$ to obtain the ensemble average brightness [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def render_pbvr(tetrahedra, camera, transfer_function, resolution, repetition_level):
    """
    Renders an irregular volume using Sorting-free Pre-integrated Projected Tetrahedra.
    
    Args:
        tetrahedra: List of tetrahedra, each with 4 vertices and scalar values at vertices.
        camera: Camera object with projection matrix and ray generation function.
        transfer_function: Function mapping scalar s -> (density rho, luminosity c).
        resolution: Tuple (width, height).
        repetition_level: Integer N, number of stochastic passes.
        
    Returns:
        Image buffer (width x height x 3) with final averaged colors.
    """
    width, height = resolution
    # Initialize accumulation buffer
    acc_buffer = [[(0.0, 0.0, 0.0) for _ in range(width)] for _ in range(height)]
    
    for rep in range(repetition_level):
        # Create a temporary buffer for this repetition
        rep_buffer = [[(0.0, 0.0, 0.0) for _ in range(width)] for _ in range(height)]
        
        # 1. Project and Rasterize
        # For each tetrahedron, project vertices to screen space.
        # Determine the 2D polygon(s) formed by the projection.
        # This involves the cell projection subdivision logic from the parent article.
        for tet in tetrahedra:
            projected_polygons = project_and_subdivide(tet, camera)
            
            for poly in projected_polygons:
                # Rasterize the polygon
                fragments = rasterize_polygon(poly, resolution)
                
                for frag in fragments:
                    # 2. Evaluate Opacity
                    # Interpolate scalar value s at fragment's 3D position within the tetrahedron
                    s = interpolate_scalar(tet, frag.world_position)
                    
                    # Map scalar to density and luminosity
                    rho, c = transfer_function(s)
                    
                    # Calculate opacity tau
                    # tau = 1 - exp(-rho * area_factor * depth_extent)
                    # In projected tetrahedra, the "area" and "length" are handled by the 
                    # projection geometry and the specific ray segment intersection.
                    # The source defines tau = 1 - exp(-rho * area * length).
                    # For a fragment, we approximate the segment properties.
                    # Let's assume the transfer function or geometry provides an effective 
                    # optical depth or we use the standard alpha from the parent article's 
                    # opacity calculation if pre-integrated.
                    # However, PBVR specifically uses the stochastic discard on the 
                    # calculated opacity.
                    
                    # Using the density emitter model opacity formula from source [S1]:
                    # tau = 1 - exp(-rho * pi * r^2 * delta_t)
                    # In the context of the fragment shader, we need the opacity for the 
                    # specific ray segment passing through the fragment.
                    # Let's assume we have a function get_segment_opacity that returns tau.
                    tau = get_segment_opacity(tet, frag, rho)
                    
                    # 3. Stochastic Discard
                    r = random.random()
                    if r <= tau:
                        # Render fragment with luminosity c
                        # Color is typically c * (1 - exp(-tau)) or similar emissive model
                        # Source says "opaque and emissive particles are used" when event occurs.
                        # We write the color to the buffer.
                        color = c # Simplified; actual color might be integrated over segment
                        rep_buffer[frag.y][frag.x] = add_color(rep_buffer[frag.y][frag.x], color)
        
        # 4. Accumulate
        for y in range(height):
            for x in range(width):
                acc_buffer[y][x] = add_color(acc_buffer[y][x], rep_buffer[y][x])
                
    # 5. Average
    final_image = [[multiply_color(acc_buffer[y][x], 1.0 / repetition_level) for x in range(width)] for y in range(height)]
    return final_image

def get_segment_opacity(tet, frag, rho):
    """
    Calculates the opacity tau for the ray segment intersecting the fragment.
    Uses the density emitter model: tau = 1 - exp(-rho * area * length).
    """
    # Determine the intersection segment of the viewing ray with the tetrahedron
    # at the fragment's screen position.
    entry_point, exit_point = ray_tet_intersection(frag.ray, tet)
    length = distance(entry_point, exit_point)
    
    # The "area" term in the source formula pi*r^2*rho*delta_t refers to the 
    # cross-section of the ray bundle. In a standard rasterization context, 
    # the fragment represents a pixel area. The opacity is usually computed 
    # per-fragment based on the integral of density along the ray.
    # tau = 1 - exp(-integral(rho ds))
    # If rho is constant in the segment: tau = 1 - exp(-rho * length)
    # The source includes an area factor pi*r^2 which relates to the particle 
    # cross-section. In volume rendering, this is often absorbed into the 
    # density definition or transfer function.
    # We assume the transfer function's 'rho' is already scaled appropriately 
    # for the volume rendering opacity equation: alpha = 1 - exp(-sigma_t * d).
    
    tau = 1.0 - math.exp(-rho * length)
    return tau

def add_color(c1, c2):
    return (c1[0] + c2[0], c1[1] + c2[1], c1[2] + c2[2])

def multiply_color(c, s):
    return (c[0] * s, c[1] * s, c[2] * s)

def project_and_subdivide(tet, camera):
    # Implementation of cell projection and subdivision from parent article
    # Returns list of 2D polygons
    pass

def rasterize_polygon(poly, resolution):
    # Returns list of fragment objects with world_position and ray
    pass

def interpolate_scalar(tet, point):
    # Linear interpolation using barycentric coordinates
    pass

def ray_tet_intersection(ray, tet):
    # Returns entry and exit points
    pass
```

**Edge Cases and Pitfalls:**
*   **Noise:** The stochastic nature introduces noise. The error decreases with the number of repetitions $N$ [S1]. Low $N$ results in visible artifacts [S1].
*   **Precipitous Transfer Functions:** PBVR may generate an insufficient number of particles (or fail to render fragments) when the density function changes drastically, leading to loss of detail in sharp iso-surfaces compared to Sorted Projected Tetrahedra (SPT) [S1].
*   **Performance Scaling:** Performance is heavily influenced by the number of pixels. Higher resolution increases the number of fragments and thus the computational cost per repetition [S1].
*   **Memory:** The method is memory-consuming as image resolution affects the number of particles/fragments processed [S1].

## Operating characteristics
*   **Time Complexity:** The deterministic brightness calculation is $O(n^2)$ if intermediate values are considered [S1]. The stochastic approach repeats the rendering $N$ times. The total time is proportional to $N \times (\text{Projection} + \text{Rasterization} + \text{Fragment Processing})$ [S1].
*   **Memory Complexity:** Memory usage scales with the number of pixels and the number of tetrahedra. It avoids the memory overhead of HAVS (which requires ~2x memory for sorting faces) [S1]. However, it is memory-consuming due to the large number of particles/fragments at high resolutions [S1].
*   **Measured Performance:** On an NVIDIA GeForce 9600GT, ~10 million particles (tetrahedra) were rendered at 25.7 milliseconds per repetition with 1024x1024 resolution [S1]. For 1.4 million tetrahedra, HAVS achieved 1.3 fps, while PBVR/SPT performance depends on $N$ [S1].
*   **Strengths:**
    *   No visibility sorting required, enabling parallelization across computing nodes for large-scale irregular volumes [S1].
    *   Avoids artifacts from insufficient k-buffer sizes in hardware-assisted sorting [S1].
    *   Handles concave sub-volumes correctly (unlike some visibility-order strategies) [S1].
    *   Better image quality than particle-based methods (no visible footprints) [S1].
*   **Limitations:**
    *   Stochastic noise requires multiple repetitions for high quality [S1].
    *   Poor handling of precipitous transfer functions compared to SPT [S1].
    *   Performance degrades significantly with high image resolution [S1].
*   **When to Use:** Large-scale irregular volumes distributed across multiple nodes, where sorting is impractical, and the transfer function is not extremely precipitous [S1].
*   **When Not to Use:** When high resolution is required and computational resources are limited, or when the transfer function has sharp discontinuities (use SPT instead) [S1].

## Background
*   **Origin:** The method was introduced in the paper "Sorting-free pre-integrated projected tetrahedra" [S1].
*   **Context:** It addresses the limitations of Particle-Based Volume Rendering (PBVR) which suffers from low image quality (visible footprints) and conventional Projected Tetrahedra (PT) which requires visibility sorting, a bottleneck for large-scale irregular volumes [S1].
*   **Development:** The technique regards volume rendering as a stochastic process, applying the density emitter model with Poisson distribution assumptions to control fragment rendering probabilistically [S1]. It is compared against Hardware-Assisted Visibility Sorting (HAVS) and Sorted Projected Tetrahedra (SPT) [S1].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
