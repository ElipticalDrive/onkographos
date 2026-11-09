# Sorting-free Pre-integrated Projected Tetrahedra

Also called: Pre-integrated Projected Tetrahedra, Sorting-free approach that controls the fragment rendering by using the opacity value, Sorting-free technique for volume rendering of an irregular volume dataset

## Overview
Sorting-free Pre-integrated Projected Tetrahedra (PBVR) is a volume rendering technique for irregular datasets that eliminates the need for global visibility sorting by treating rendering as a stochastic process [S1]. Unlike conventional projected tetrahedra which require back-to-front compositing, PBVR controls fragment rendering probabilistically based on opacity, allowing correct semi-transparent image synthesis without explicit cell ordering [S1]. It improves upon particle-based methods by avoiding visible footprints and upon sorting-based methods by removing scalability bottlenecks associated with large-scale irregular volumes [S1].

## How it works
**Inputs:** Irregular volume dataset decomposed into tetrahedral cells; transfer function mapping scalar values to density ($\rho$) and luminosity ($c$); camera parameters.
**Outputs:** Final image brightness/color.

**Data Structures:**
*   **Tetrahedral Mesh:** List of tetrahedra, each defined by 4 vertices and associated scalar values at vertices.
*   **Projected Polygons:** For each tetrahedron, the 2D screen-space projection is computed. The projected edges divide the image plane into polygons. Each polygon corresponds to the projection of a specific ray segment passing through the tetrahedron [S1][S4, book].
*   **Segment Parameters:** For each polygon, the average differential opacity (extinction coefficient) $\tau_{avg}$ and the segment length $l$ are computed. $\tau_{avg}$ is interpolated from cell data; $l$ is interpolated from vertex depths [S1][S4, book].

**Core Optical Model:**
The method uses the density emitter model, assuming the number of particles in a unit volume follows a Poisson distribution [S1]. The opacity $\alpha$ for a ray segment of length $l$ with average density $\rho$ is derived from this distribution:
$$ \alpha = 1 - e^{-\rho \cdot \text{area} \cdot l} $$
In the context of projected tetrahedra, the "area" term is absorbed into the density definition or normalized, such that the effective opacity depends on $\rho$ and $l$ [S1]. Specifically, the text defines $\pi r^2 \rho \Delta t$ as the number of particles in a segment, where $\Delta t$ is the segment length [S1].

**Algorithm Steps:**
1.  **Pre-integration:** Compute a 2D texture lookup table for opacity $\alpha(\tau_{avg}, l)$ using the formula $\alpha = 1 - \exp(-\tau_{avg} l)$ to avoid Mach bands from linear interpolation [S1][S4, book].
2.  **Projection:** Project all tetrahedral vertices to screen space. Generate the set of polygons representing the cell projections.
3.  **Rasterization and Stochastic Sampling:** For each polygon:
    a.  Rasterize the polygon.
    b.  For each fragment, compute the local opacity $\alpha$ using the pre-integrated texture lookup with coordinates $(\tau_{avg}, l)$ [S1].
    c.  Generate a random number $r \in [0, 1]$.
    d.  If $r > \alpha$, discard the fragment.
    e.  If $r \le \alpha$, render the fragment with its associated luminosity/color [S1].
4.  **Repetition:** Repeat steps 3a-3e for $N$ repetitions (repetition level) [S1].
5.  **Accumulation:** Average the pixel values from all $N$ repetitions to obtain the final brightness [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pbvr(tetrahedra, transfer_function, camera, num_repetitions, image_width, image_height):
    # Initialize accumulator image
    accumulator = zeros((image_height, image_width, 3))
    
    # Pre-compute opacity lookup table if not already done
    # opacities = preintegrate_opacity_table() 
    
    for rep in range(num_repetitions):
        # Initialize current frame image
        current_frame = zeros((image_height, image_width, 3))
        
        # Process each tetrahedron
        for tet in tetrahedra:
            # Project vertices to screen space
            projected_vertices = project_vertices(tet.vertices, camera)
            
            # Generate polygons from projected edges
            # This step involves clipping and decomposing the projected tetrahedron
            # into convex polygons in screen space.
            polygons = decompose_projected_tetrahedron(projected_vertices)
            
            for poly in polygons:
                # Compute average density and segment length for the polygon
                # These are interpolated from the tetrahedron's vertex data
                avg_density = interpolate_density(poly, tet)
                segment_length = interpolate_length(poly, tet)
                
                # Compute opacity using pre-integration or direct formula
                # Using direct formula for clarity, though texture lookup is preferred
                opacity = 1.0 - exp(-avg_density * segment_length)
                
                # Rasterize polygon
                fragments = rasterize_polygon(poly)
                
                for frag in fragments:
                    # Generate random number for stochastic sampling
                    r = random_float(0.0, 1.0)
                    
                    # Stochastic acceptance/rejection
                    if r <= opacity:
                        # Get color/luminosity from transfer function
                        color = transfer_function(avg_density)
                        
                        # Add to current frame (simple accumulation for averaging later)
                        # Note: In hardware, this might be done via alpha blending 
                        # if treating the stochastic event as an opaque object, 
                        # but the source describes averaging pixel values.
                        current_frame[frag.y, frag.x] += color
        
        # Accumulate results
        accumulator += current_frame
    
    # Average over repetitions
    final_image = accumulator / num_repetitions
    return final_image
```

**Parameters:**
*   **Repetition Level ($N$):** The number of times the rendering calculation is repeated. Higher $N$ reduces error but increases computation time. Typical values tested: 1, 25, 100, 144 [S1].
*   **Resolution:** Image resolution (e.g., 1024x1024) affects the number of pixels and thus the total number of particles/samples [S1].

**Edge Cases and Pitfalls:**
*   **Drastic Density Changes:** PBVR may generate an insufficient number of particles when the density function changes drastically, leading to artifacts compared to sorting-based techniques like SPT [S1].
*   **Memory Consumption:** Performance is greatly influenced by the number of pixels. High resolution increases memory consumption because it affects the number of particles processed [S1].
*   **Convergence:** Low repetition levels result in noisy images. The error decreases with the number of repetitions [S1].

## Operating characteristics
*   **Time Complexity:** The basic brightness calculation without stochastic approximation is $O(n^2)$ if intermediate brightness values are considered [S1]. PBVR's performance is influenced by the number of cells and the number of pixels. It is memory-consuming as image resolution affects the number of particles [S1].
*   **Measured Performance:** On an NVIDIA GeForce 9600GT, PBVR rendered ~10 million particles at 25.7 milliseconds per repetition with 1024x1024 resolution [S1].
*   **Strengths:** No visibility sorting required, making it scalable for large-scale irregular volumes distributed across multiple nodes. Avoids artifacts from insufficient k-buffer sizes in hardware-assisted sorting. No visible footprints when the viewpoint is inside the volume (unlike particle-based methods) [S1].
*   **Limitations:** High memory usage at high resolutions. Noisy images at low repetition levels. Less effective than sorting-based methods (SPT) for precipitous transfer functions [S1].
*   **When to Use:** Large-scale irregular volumes where visibility sorting is impractical or impossible, and high resolution is not the primary constraint.
*   **When Not to Use:** When precise visualization of sharp density gradients is required (SPT is better), or when memory is severely constrained at high resolutions [S1].

## Background
PBVR was introduced as a sorting-free technique for volume rendering of irregular volume datasets [S1]. It addresses the limitations of particle-based volume rendering (low image quality, visible footprints) and conventional projected tetrahedra (requirement for visibility sorting, scalability issues) [S1]. The method is based on the density emitter model and the Poisson distribution of particles [S1]. It was developed to handle large-scale simulation results (CSM and CFD) that are divided into multiple sub-volumes across computing nodes, where global visibility sorting is impractical [S1].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
