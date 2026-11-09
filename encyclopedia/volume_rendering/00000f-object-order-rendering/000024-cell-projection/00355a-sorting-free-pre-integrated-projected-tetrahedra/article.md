# Sorting-free Pre-integrated Projected Tetrahedra

Also called: Pre-integrated Projected Tetrahedra, Sorting-free approach that controls the fragment rendering by using the opacity value, Sorting-free technique for volume rendering of an irregular volume dataset

## Overview
Sorting-free Pre-integrated Projected Tetrahedra (PBVR) is a volume rendering technique for irregular datasets that eliminates the need for visibility sorting by treating rendering as a stochastic process [S1]. It solves the scalability issues of conventional sorting-based methods and the low image quality (visible footprints) of particle-based methods by employing projected tetrahedra with pre-integration [S1]. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which requires back-to-front sorting, PBVR achieves semi-transparency through multiple rendering repetitions controlled by opacity probabilities [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An irregular volume dataset decomposed into tetrahedral cells, a transfer function mapping scalar values to density ($\rho$) and luminosity ($c$), and a camera/viewing ray definition.
*   **Output:** A rendered image representing the ensemble average of brightness values over $N$ repetitions [S1].

**Data Structures:**
The method operates on tetrahedral cells. Each cell is treated as a set of polygons (faces) for rasterization. The core data structure for the stochastic model is the ray segment, defined by:
*   $k$: Segment index.
*   $\Delta t_k$: Length of the $k$-th ray segment within the tetrahedron.
*   $\rho_k$: Particle density (number of particles per unit volume) in the segment.
*   $c_k$: Particle luminosity in the segment.
*   $r$: Effective particle radius (used in the cross-sectional area calculation) [S1].

**Mathematical Model:**
The method uses a **density emitter model** where the 3D scalar field is characterized as a varying density emitter with single-level scattering [S1]. The number of particles in a volume follows a Poisson distribution [S1].

1.  **Opacity Calculation:**
    The opacity $\tau$ for a ray segment of length $\Delta t$ with density $\rho$ is derived from the probability that the segment contains particles. The expected number of particles in the segment is $\pi r^2 \rho \Delta t$ [S1]. The opacity is defined as:
    $$ \tau = 1 - e^{-\pi r^2 \rho \Delta t} $$
    [S1]. Note: In the context of the source, this $\tau$ represents the probability of interaction/emission for the stochastic step.

2.  **Brightness Calculation (Deterministic Baseline):**
    The brightness $B_k$ for a segment is calculated as:
    $$ B_k = c_k \times (1 - e^{-\pi r^2 \rho \Delta t}) $$
    [S1]. This represents the expected value of luminosity from the ray segment.

3.  **Stochastic Rendering (Sorting-Free Approach):**
    To avoid sorting, the rendering is repeated $N$ times. In each repetition, fragments are rendered with a probability equal to the opacity value $\tau$ at each ray segment [S1].
    *   **Step 1:** For each ray segment along the viewing ray, generate a random number $u \in [0, 1]$.
    *   **Step 2:** If $u < \tau$, the fragment is rendered (opaque and emissive); otherwise, it is discarded [S1].
    *   **Step 3:** The luminosity of the rendered fragment is $c_k$ [S1].
    *   **Step 4:** Repeat Steps 1-3 for $N$ repetitions.
    *   **Step 5:** The final pixel value is the ensemble average (mean) of the brightness values accumulated over all $N$ repetitions [S1].

    The probability that the $k$-th segment contributes to the brightness (i.e., no particles in previous segments and $\ge 1$ particle in current segment) is given by:
    $$ P_k = \left( \prod_{i=1}^{k-1} (1 - \tau_i) \right) \tau_k $$
    [S1]. This formulation ensures that the stochastic accumulation approximates the deterministic back-to-front integration without explicit sorting.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pbvr(tetrahedra, transfer_function, camera, num_repetitions):
    """
    Renders an irregular volume using Sorting-free Pre-integrated Projected Tetrahedra.
    
    Args:
        tetrahedra: List of tetrahedral cells.
        transfer_function: Function mapping scalar value -> (density rho, luminosity c).
        camera: Camera object with ray generation capability.
        num_repetitions: N, the number of stochastic repetitions.
        
    Returns:
        image: Rendered image buffer.
    """
    # Initialize image buffer to zero
    image = zeros(camera.resolution)
    
    for repetition in range(num_repetitions):
        # Create a temporary buffer for this repetition
        frame_buffer = zeros(camera.resolution)
        
        # Process each tetrahedron (Object-order rendering)
        for tet in tetrahedra:
            # 1. Intersect viewing rays with the tetrahedron
            # Get list of ray segments intersecting this tet
            segments = intersect_tetrahedron(tet, camera.rays)
            
            for segment in segments:
                # 2. Evaluate scalar value at segment (e.g., centroid or barycentric interp)
                scalar_val = evaluate_scalar(tet, segment.position)
                
                # 3. Get density and luminosity from transfer function
                rho, c = transfer_function(scalar_val)
                
                # 4. Calculate opacity (probability of interaction)
                # Using particle radius r (constant or derived from transfer function)
                # Area = pi * r^2
                area = pi * r_squared
                opacity = 1.0 - exp(-area * rho * segment.length)
                
                # 5. Stochastic decision
                # Generate random number u in [0, 1]
                u = random_float()
                
                if u < opacity:
                    # Fragment is rendered
                    # Calculate brightness contribution
                    brightness = c * opacity # Expected value approximation for the segment
                    # Add to frame buffer (simple accumulation for averaging later)
                    # Note: In a real shader, this would be a fragment shader operation
                    add_to_pixel(frame_buffer, segment.screen_coords, brightness)
        
        # Accumulate results from this repetition
        # Since we want the ensemble average, we can sum all frames and divide by N at the end
        # Or maintain a running average. Here we sum.
        image += frame_buffer
        
    # Final ensemble average
    image = image / num_repetitions
    
    return image

def intersect_tetrahedron(tet, rays):
    """
    Returns list of ray segments intersecting the tetrahedron.
    Each segment has: position, length, screen_coords.
    """
    segments = []
    for ray in rays:
        # Standard ray-tet intersection
        hits = ray_intersect_tet(ray, tet)
        if hits:
            for hit in hits:
                segments.append({
                    'position': hit.point,
                    'length': hit.distance,
                    'screen_coords': ray.project(hit.point)
                })
    return segments

def evaluate_scalar(tet, position):
    """
    Linearly interpolate scalar value from tetrahedron vertices using barycentric coordinates.
    """
    # Barycentric interpolation
    s = interpolate_barycentric(tet.vertices, tet.scalars, position)
    return s
```

**Parameters:**
*   $N$ (Repetition Level): The number of times the rendering calculation is repeated. Higher $N$ reduces error but increases computation time [S1]. Typical values tested: 1, 25, 100, 144 [S1].
*   $r$ (Particle Radius): A parameter in the density emitter model. Not explicitly defined in the source text as a variable input, but implied as part of the cross-sectional area $\pi r^2$ [S1].

**Edge Cases and Pitfalls:**
*   **Drastic Density Changes:** PBVR may generate an insufficient number of particles (or fail to sample correctly) when the density function changes drastically, leading to artifacts compared to [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md) (SPT) which handles precipitous transfer functions better [S1].
*   **Memory Consumption:** Performance is greatly influenced by the number of pixels. Increasing image resolution affects the number of particles/ray segments processed, making it memory-consuming [S1].
*   **Convergence:** The error decreases with the number of repetitions $N$. Insufficient $N$ leads to noisy images [S1].

## Operating characteristics
*   **Time Complexity:** The basic deterministic brightness calculation has a complexity of $O(n^2)$ if intermediate brightness values are considered [S1]. The stochastic approach requires $N$ repetitions, so the total time is proportional to $N \times (\text{ray-tet intersection cost} + \text{rasterization cost})$.
*   **Memory Complexity:** Memory usage is driven by the number of pixels and the need to store intermediate buffers for $N$ repetitions (or accumulate them). It is described as memory-consuming due to the impact of image resolution on particle count [S1].
*   **Measured Performance:** On an NVIDIA GeForce 9600GT, PBVR rendered ~10 million particles at 1024x1024 resolution in 25.7 milliseconds per repetition [S1]. On an Intel Core 2 Quad 2.83 GHz with NVIDIA GeForce GTX 280, it handled large-scale CSM and CFD simulation results [S1].
*   **Strengths:**
    *   No visibility sorting required, making it suitable for large-scale irregular volumes distributed across multiple computing nodes [S1].
    *   Avoids artifacts associated with insufficient k-buffer sizes in Hardware-Assisted Visibility Sorting (HAVS) [S1].
    *   Better image quality than particle-based methods (no visible footprints) [S1].
*   **Limitations:**
    *   Performance is heavily dependent on image resolution (number of pixels) [S1].
    *   Struggles with precipitous transfer functions compared to sorting-based methods like SPT [S1].
    *   Requires multiple repetitions to achieve high-quality semi-transparency, increasing compute time [S1].
*   **When to Use:** Large-scale irregular volumes where visibility sorting is impractical or impossible (e.g., distributed computing), and when particle-based artifacts are unacceptable.
*   **When Not to Use:** When high performance is needed for small datasets, or when the transfer function has sharp discontinuities that require precise sorting (use SPT instead) [S1].

## Background
*   **Origin:** The method was introduced in the paper "Sorting-free pre-integrated projected tetrahedra" [S1].
*   **History:** It was developed to address the limitations of Particle-Based Volume Rendering (PBVR) which suffers from low image quality (visible footprints) and conventional Projected Tetrahedra (PT) which requires visibility sorting [S1]. It builds on the density emitter model [S1].
*   **Usage:** Applied to large-scale irregular volumes from CSM (Computational Solid Mechanics) and CFD (Computational Fluid Dynamics) simulations [S1].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
