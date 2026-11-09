# Sorting-free Pre-integrated Projected Tetrahedra

Also called: Pre-integrated Projected Tetrahedra, Sorting-free approach that controls the fragment rendering by using the opacity value, Sorting-free technique for volume rendering of an irregular volume dataset

## Overview
Sorting-free Pre-integrated Projected Tetrahedra (PBVR) is a volume rendering technique for irregular volume datasets that eliminates the need for explicit back-to-front visibility sorting of tetrahedral cells [S1]. It solves the scalability bottlenecks of conventional sorting-based methods and the low image quality (visible footprints) of particle-based methods by treating volume rendering as a stochastic process where sampling points occur along a viewing ray with a probability equal to the local opacity [S1]. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which requires visibility sorting, PBVR achieves semi-transparency through repeated rasterization passes controlled by opacity probabilities, avoiding the memory overhead and artifacts associated with hardware-assisted sorting like [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An irregular volume dataset decomposed into tetrahedral cells. Each cell is defined by 4 vertices with associated scalar values. The scalar field is assumed to vary linearly across the cell [S1, S2].
*   **Output:** A rendered 2D image representing the integrated brightness/color of the volume.

**Data Structures:**
*   **Tetrahedral Mesh:** A set of tetrahedra, each defined by 4 vertices. Each vertex holds 3D coordinates and a scalar value $s$ [S1, S2].
*   **Pre-integrated Texture (Optional but implied by name):** While the core PBVR algorithm described in the primary source focuses on the stochastic sampling, the method name "Pre-integrated" implies the use of pre-computed integration tables for opacity/color transfer, similar to [Projected Tetrahedra](877-projected-tetrahedra.md). However, the specific pre-integration formulas are not detailed in the provided notes for PBVR itself, only for the general cell projection family [S1, S2]. The PBVR specific contribution is the sorting-free stochastic compositing.

**Core Optical Model:**
The method uses a density emitter model where the 3D scalar field is characterized as a varying density emitter with a single level of scattering [S1]. The opacity is derived from the number of particles in a unit volume, assuming the number of particles follows a Poisson distribution [S1].

**Numbered Steps:**

1.  **Projection and Rasterization:**
    For each tetrahedral cell, project the 3D vertices to 2D screen space. Subdivide the projected cell into polygons based on the projected edges, ensuring each polygon lies within the projection of a single front-facing face and a single back-facing face [S1, S2]. Rasterize these polygons to generate fragments [S1].

2.  **Scalar Interpolation and Opacity Calculation:**
    For each fragment, determine the corresponding ray segment within the tetrahedron. Calculate the scalar value $s$ at the fragment's position using linear interpolation from the vertex scalar values [S1, S2]. Map this scalar value to a differential opacity $\tau$ or particle density $\rho$ using a transfer function [S1, S2].
    The opacity $\alpha$ for a ray segment of length $\Delta t$ is calculated based on the Poisson distribution assumption:
    $$ \alpha = 1 - e^{-\rho \cdot \pi r^2 \cdot \Delta t} $$
    where $\rho$ is the number of particles in a unit volume, and $\pi r^2 \Delta t$ expresses the number of particles in the ray segment [S1]. Note: In the context of the stochastic approach, this $\alpha$ represents the probability of a "particle" (light contribution) being present in that segment.

3.  **Stochastic Fragment Rendering (The Sorting-Free Step):**
    Instead of sorting fragments or cells, render the fragments multiple times. In each repetition $n$ (from 1 to $N$, where $N$ is the repetition level):
    *   For each fragment generated in Step 2, generate a random number $r \in [0, 1]$.
    *   Compare $r$ with the calculated opacity $\alpha$ for that fragment.
    *   If $r < \alpha$, the fragment is rendered with its full luminosity/color.
    *   If $r \ge \alpha$, the fragment is discarded (not rendered) for this repetition [S1].

4.  **Accumulation:**
    After $N$ repetitions, accumulate the rendered pixel values from all repetitions. The final pixel value is the average of the brightness values obtained from the $N$ repetitions [S1]. This averaging approximates the expected value of the luminosity, effectively achieving semi-transparency without alpha-blending or sorting [S1].

**Formulas:**
*   **Brightness of a segment:** $B_k = c_k \times (1 - e^{-\rho \cdot \text{area} \cdot \text{length}})$ [S1]. Here $c_k$ is particle luminosity.
*   **Opacity (Probability):** $\alpha = 1 - e^{-\rho \cdot \pi r^2 \cdot \Delta t}$ [S1].
*   **Probability of Event:** The possibility that the $k$-th luminosity is equal to the brightness value is defined by the event where there are no particles from the first to the $(k-1)$-th ray-segments and more than one particle in the $k$-th ray segment [S1].
*   **Complexity:** The computational complexity of the basic brightness calculation is $O(n^2)$ if intermediate brightness values are considered [S1].

**Parameters:**
*   **Repetition Level ($N$):** The number of times the rendering calculation is repeated. Higher $N$ reduces error but increases computation time. Typical values tested: 1, 25, 100, 144 [S1].
*   **Resolution:** Image resolution (e.g., 1024x1024) significantly impacts performance and memory usage [S1].

**Edge Cases and Pitfalls:**
*   **Drastic Density Changes:** PBVR may generate an insufficient number of particles when the density function changes drastically, leading to artifacts compared to methods like [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) or SPT which handle precipitous transfer functions better [S1].
*   **Memory Consumption:** PBVR is memory-consuming because the increase in image resolution affects the number of particles processed [S1].
*   **Performance Scaling:** Performance is greatly influenced by the number of pixels. For large-scale irregular volumes, PBVR is preferred over sorting-based methods if the number of particles/cells exceeds ~24 million [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pbvr(tetrahedra, camera, resolution, repetition_level):
    """
    Renders an irregular volume using Sorting-free Pre-integrated Projected Tetrahedra.
    
    Args:
        tetrahedra: List of tetrahedra, each with 4 vertices (pos, scalar).
        camera: Camera object with projection matrix and view direction.
        resolution: Tuple (width, height).
        repetition_level: Integer N, number of stochastic repetitions.
        
    Returns:
        Image buffer (width x height x 3).
    """
    width, height = resolution
    image_accumulator = np.zeros((height, width, 3))
    
    # Pre-calculate projection data for all tetrahedra
    # This involves projecting vertices and subdividing into polygons
    # as described in the parent article [Projected Tetrahedra](877-projected-tetrahedra.md)
    projected_polys = []
    for tet in tetrahedra:
        polys = project_and_subdivide(tet, camera)
        projected_polys.extend(polys)
        
    for rep in range(repetition_level):
        current_frame = np.zeros((height, width, 3))
        
        for poly in projected_polys:
            # Rasterize the polygon to get fragments
            fragments = rasterize(poly, width, height)
            
            for frag in fragments:
                # Calculate opacity for this fragment's ray segment
                # Interpolate scalar value from tetrahedron vertices
                scalar_val = interpolate_scalar(frag.position, tet)
                
                # Map scalar to density/rho using transfer function
                rho = transfer_function(scalar_val)
                
                # Calculate opacity/probability alpha
                # alpha = 1 - exp(-rho * pi * r^2 * delta_t)
                # Note: r and delta_t depend on ray segment geometry
                alpha = 1.0 - math.exp(-rho * math.pi * frag.ray_radius**2 * frag.ray_length)
                
                # Stochastic sampling
                random_val = random.uniform(0.0, 1.0)
                
                if random_val < alpha:
                    # Fragment is "active" for this repetition
                    # Calculate brightness/color
                    # B = c_k * (1 - exp(...))
                    color = calculate_color(scalar_val, rho, frag.ray_length)
                    current_frame[frag.y, frag.x] += color
                    
        # Accumulate the result of this repetition
        image_accumulator += current_frame
        
    # Average over all repetitions
    final_image = image_accumulator / repetition_level
    
    return final_image

def project_and_subdivide(tet, camera):
    """
    Projects a tetrahedron to 2D and subdivides it into polygons.
    Returns a list of polygons with associated tetrahedron reference.
    """
    # Project 4 vertices to screen space
    vertices_2d = [project_vertex(v, camera) for v in tet.vertices]
    
    # Subdivide the projected area into polygons based on projected edges
    # This ensures each polygon lies within a single front/back face projection
    # Implementation details follow the incremental method or lookup tables
    # from the parent article [Projected Tetrahedra](877-projected-tetrahedra.md)
    polygons = []
    # ... (Implementation of subdivision logic) ...
    return polygons

def interpolate_scalar(position, tet):
    """
    Interpolates scalar value at position within tetrahedron.
    """
    # Barycentric coordinates or linear interpolation along edges
    # s = sum(s_i * lambda_i)
    return 0.0 # Placeholder for interpolation logic

def transfer_function(scalar):
    """
    Maps scalar value to particle density rho.
    """
    return 0.0 # Placeholder for transfer function

def calculate_color(scalar, rho, length):
    """
    Calculates the color/brightness contribution.
    """
    return 0.0 # Placeholder for color calculation
```

## Operating characteristics
*   **Time Complexity:** The basic brightness calculation has a complexity of $O(n^2)$ if intermediate brightness values are considered [S1]. The overall performance is heavily influenced by the number of pixels and the repetition level $N$ [S1].
*   **Memory Complexity:** PBVR is memory-consuming because the increase in image resolution affects the number of particles [S1]. It requires less memory than Hardware-Assisted Visibility Sorting (HAVS), which needs memory space for sorting cell faces about twice that for tetrahedral cells [S1].
*   **Measured Performance:** On an NVIDIA GeForce 9600GT, PBVR rendered ~10 million particles at 25.7 milliseconds per repetition with 1024x1024 resolution [S1]. For 1024x1024 resolution, the computing time is estimated at 1.4 seconds per repetition for large datasets [S1].
*   **Strengths:**
    *   No visibility sorting required, making it suitable for large-scale irregular volumes distributed across multiple computing nodes [S1].
    *   Avoids artifacts caused by insufficient k-buffer size in HAVS [S1].
    *   Avoids visible footprints when the viewpoint is inside the volume, unlike particle-based methods [S1].
    *   Can handle larger irregular volume datasets than sorting-based methods if the number of particles exceeds ~24 million [S1].
*   **Limitations:**
    *   Performance is greatly influenced by the number of pixels [S1].
    *   May generate insufficient particles when density functions change drastically, leading to lower quality compared to SPT for precipitous transfer functions [S1].
    *   Requires multiple repetitions to achieve high-quality images, increasing computation time [S1].
*   **When to Use:** Use PBVR for large-scale irregular volume datasets where visibility sorting is a bottleneck or impractical (e.g., distributed computing), and when the number of particles/cells is large (>24 million). It is preferred over HAVS when memory is constrained and over PBVR (particle-based) when viewpoint is inside the volume.
*   **When Not to Use:** Avoid PBVR if the density function changes drastically and high-quality visualization of precipitous transfer functions is required (use SPT instead) [S1]. Avoid if real-time performance is critical and the dataset is small, as the repetition overhead may be prohibitive.

## Background
*   **Origin:** The method was introduced in the paper "Sorting-free pre-integrated projected tetrahedra" [S1].
*   **History:** The technique was developed to address the limitations of particle-based volume rendering (low image quality, visible footprints) and conventional projected tetrahedra (requirement for visibility sorting, scalability issues with large irregular volumes) [S1]. It builds on the density emitter model and Poisson distribution assumptions for particle counts [S1].
*   **Usage:** It is used for rendering large-scale irregular volume datasets, such as those from CSM (Computational Solid Mechanics) and CFD (Computational Fluid Dynamics) simulations [S1].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
