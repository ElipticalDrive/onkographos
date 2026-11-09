# Sorting-free Pre-integrated Projected Tetrahedra

Also called: Pre-integrated Projected Tetrahedra, Sorting-free approach that controls the fragment rendering by using the opacity value, Sorting-free technique for volume rendering of an irregular volume dataset

## Overview
Sorting-free Pre-integrated Projected Tetrahedra (PBVR) is a stochastic volume rendering technique for irregular datasets that eliminates the need for explicit back-to-front visibility sorting [S1]. It solves the scalability bottlenecks of conventional sorting methods and the visible footprint artifacts of particle-based rendering by treating the rendering process as a stochastic sampling event controlled by opacity [S1]. Unlike standard Projected Tetrahedra, which requires sorting cells to composite correctly, PBVR renders fragments multiple times, discarding them based on a probability check against their opacity, thereby achieving correct semi-transparent compositing without sorting [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An irregular volume dataset decomposed into tetrahedral cells, each associated with scalar values at vertices. A transfer function mapping scalar values to color and extinction (opacity).
*   **Output:** A 2D rendered image.

**Data Structures:**
*   **Tetrahedral Mesh:** Defined by vertices and connectivity. As per the parent article [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md), cells are projected to 2D screen space, and edges subdivide the image plane into polygons [S1][S2][S4].
*   **Pre-integration Table:** Since the method employs "pre-integrated" tetrahedra, it relies on a lookup table (typically a 3D texture) that stores pre-computed optical properties (color and opacity) for segments of varying lengths and scalar values, derived from the transfer function [S1].

**Core Optical Model:**
The method uses a density emitter model where the scalar field is characterized as a varying density emitter with single-level scattering [S1]. The opacity is derived from the number of particles in a unit volume, assuming the particle count follows a Poisson distribution [S1].

**Numbered Steps:**
1.  **Projection and Rasterization:** Project the 3D tetrahedral vertices to 2D screen space. Rasterize the projected faces to generate fragments [S1].
2.  **Opacity Evaluation:** For each fragment, evaluate the opacity $\alpha$ based on the pre-integrated data or the density model [S1].
3.  **Stochastic Sampling (The Sorting-Free Mechanism):**
    *   The rendering calculation is repeated $N$ times (the repetition level) [S1].
    *   In each repetition, for every fragment, generate a random number $r$ uniformly distributed in $[0, 1]$ [S1].
    *   Compare $r$ with the fragment's opacity $\alpha$ [S1].
    *   If $r > \alpha$, discard the fragment (treat it as transparent/empty) [S1].
    *   If $r \le \alpha$, render the fragment with its associated color/brightness [S1].
4.  **Accumulation:** Accumulate the results from all $N$ repetitions. The final pixel value is the average of the brightness values obtained from the $N$ repetitions [S1].

**Formulas:**
*   **Brightness of a Ray Segment:**
    The brightness $B_k$ for the $k$-th ray segment is calculated as:
    $$ B_k = c_k \times (1 - e^{-\rho \times \text{area} \times \Delta t}) $$
    Where:
    *   $c_k$ is the particle luminosity in the $k$-th segment [S1].
    *   $\rho$ is the particle density (number of particles in a unit volume) [S1].
    *   $\Delta t$ is the ray-segment length [S1].
    *   $\text{area}$ refers to the cross-sectional area of the ray segment (implied as $\pi r^2$ in the context of particle count $\pi r^2 \rho \Delta t$) [S1].
    *   Note: The text notes that if intermediate brightness values $B_k$ are considered for back-to-front accumulation, the complexity is $O(n^2)$ [S1]. However, PBVR avoids this by using the stochastic approach [S1].

*   **Opacity (Density Emitter Model):**
    $$ \tau = 1 - e^{-\rho \times \text{area} \times \Delta t} $$
    This represents the probability that the segment contains some particles, based on the Poisson distribution assumption [S1].

*   **Probability of Luminosity Event:**
    The possibility that the $k$-th luminosity is equal to the brightness value is defined by the event where there are no particles from the first to the $(k-1)$-th ray segments, and more than one particle in the $k$-th segment [S1].

**Parameters:**
*   **Repetition Level ($N$):** The number of times the rendering calculation is repeated. Higher $N$ reduces error but increases computation time [S1]. Typical values tested include 1, 25, 100, and 144 [S1].
*   **Resolution:** Performance is heavily influenced by the number of pixels. Experiments used 1024x1024 resolution [S1].

**Edge Cases and Pitfalls:**
*   **Drastic Density Changes:** PBVR may generate an insufficient number of particles when the density function changes drastically, leading to artifacts compared to Sorting-free Projected Tetrahedra (SPT) [S1].
*   **Memory Consumption:** PBVR is memory-consuming because the increase in image resolution affects the number of particles required for accurate stochastic sampling [S1].
*   **Convergence:** Low repetition levels ($N=1$) result in noisy images; sufficient repetitions are needed to approximate the expected value of luminosity [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pbvr(tetrahedra, transfer_function, resolution, repetition_level):
    """
    Renders an irregular volume using Sorting-free Pre-integrated Projected Tetrahedra.
    
    Args:
        tetrahedra: List of tetrahedral cells with vertex positions and scalar values.
        transfer_function: Function mapping scalar values to color and opacity.
        resolution: Tuple (width, height) of the output image.
        repetition_level: Integer N, the number of stochastic repetitions.
        
    Returns:
        Image buffer with accumulated brightness values.
    """
    # Initialize image buffer to zero
    image = zeros(resolution)
    
    # Pre-compute projected polygons for all tetrahedra
    # This step relies on the parent article's projection logic
    projected_faces = project_tetrahedra(tetrahedra, resolution)
    
    for rep in range(repetition_level):
        # Create a temporary buffer for this repetition
        temp_image = zeros(resolution)
        
        for face in projected_faces:
            # Rasterize the face to get fragments
            fragments = rasterize(face)
            
            for frag in fragments:
                # Evaluate opacity based on the scalar value at the fragment's position
                # Using pre-integration or the density model
                opacity = evaluate_opacity(frag.scalar_value, transfer_function)
                
                # Stochastic sampling: discard fragment with probability (1 - opacity)
                random_val = random_uniform(0, 1)
                
                if random_val <= opacity:
                    # Fragment is "visible" in this stochastic sample
                    # Retrieve color/brightness from pre-integration table or model
                    brightness = evaluate_brightness(frag.scalar_value, transfer_function)
                    
                    # Accumulate brightness
                    temp_image[frag.x, frag.y] += brightness
        
        # Average the result of this repetition into the final image
        # Note: The source describes accumulating brightness values and then averaging
        # across repetitions.
        for x in range(resolution[0]):
            for y in range(resolution[1]):
                image[x, y] += temp_image[x, y]
                
    # Final averaging
    image = image / repetition_level
    
    return image

def project_tetrahedra(tetrahedra, resolution):
    """
    Projects 3D tetrahedra to 2D screen space.
    Implements the logic from [Tetrahedral representation and traversal](4396-tetrahedral-representation-and-traversal.md).
    """
    projected = []
    for tet in tetrahedra:
        # Project vertices
        projected_vertices = [project_vertex(v, resolution) for v in tet.vertices]
        
        # Determine visible faces (front/back facing)
        # Subdivide the image plane based on projected edges
        faces = extract_faces(projected_vertices, tet)
        
        for face in faces:
            if is_front_facing(face):
                projected.append(face)
    return projected

def evaluate_opacity(scalar, transfer_function):
    """
    Evaluates opacity using the density emitter model or pre-integration.
    """
    # Using the density emitter model formula from [S1]
    # tau = 1 - exp(-rho * area * delta_t)
    # In the context of fragment rendering, this is mapped via the transfer function
    return transfer_function.get_opacity(scalar)

def evaluate_brightness(scalar, transfer_function):
    """
    Evaluates brightness/color for the fragment.
    """
    return transfer_function.get_color(scalar)
```

## Subtopics

### Projected Tetrahedra
The base method that PBVR modifies. It requires visibility sorting. [Projected Tetrahedra](877-projected-tetrahedra.md)

### Parallelized Accurate Sorting for Projected Tetrahedra
A sibling method that uses hardware-assisted visibility sorting (HAVS) to avoid software sorting bottlenecks but still requires sorting. [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)

### Sorting-free Projected Tetrahedra (SPT)
A related sorting-free technique that handles precipitous transfer functions better than PBVR but may suffer from insufficient particle generation in drastic density changes. [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md) (Note: SPT is mentioned in comparison but its specific article link is not explicitly provided in the sibling list, so it is linked to the closest relevant sorting-free context or the parent if no specific SPT article exists. However, the prompt lists "Parallelized Accurate Sorting..." as a sibling. SPT is distinct from PBVR. If no specific SPT article is in the tree, we link to the parent or note it. The prompt asks to link to other articles. SPT is not in the sibling list. I will link to the parent or omit if no link exists. The prompt says "Links to other articles use the file names given below". SPT is not in the list. I will not invent a link.)

## Operating characteristics
*   **Time Complexity:** The basic brightness calculation with intermediate values is $O(n^2)$ [S1]. PBVR's performance is greatly influenced by the number of pixels and the repetition level $N$ [S1].
*   **Memory Complexity:** PBVR is memory-consuming because the number of particles (and thus memory for stochastic sampling) increases with image resolution [S1].
*   **Measured Performance:** On an NVIDIA GeForce 9600GT, PBVR rendered ~10 million particles at 1024x1024 resolution in 25.7 milliseconds per repetition [S1].
*   **Strengths:**
    *   No visibility sorting required, making it suitable for large-scale irregular volumes distributed across multiple computing nodes [S1].
    *   Avoids artifacts caused by insufficient k-buffer size associated with HAVS [S1].
    *   Avoids visible footprints when the viewpoint is inside the volume, unlike particle-based methods [S1].
*   **Limitations:**
    *   Performance degrades with higher image resolutions due to increased particle count requirements [S1].
    *   May produce insufficient particles and artifacts when density functions change drastically, compared to SPT [S1].
    *   Requires multiple repetitions to achieve high-quality images, increasing computation time [S1].
*   **When to Use:** Large-scale irregular volume datasets where visibility sorting is impractical or impossible (e.g., distributed computing environments) and where the transfer function does not change drastically [S1].
*   **When Not to Use:** When high performance is required for small datasets (HAVS may be faster) or when the density function changes drastically (SPT may be better) [S1].

## Background
*   **Origin:** The method was introduced in the paper "Sorting-free pre-integrated projected tetrahedra" [S1].
*   **History:** It was developed to address the limitations of particle-based volume rendering (low image quality, visible footprints) and conventional projected tetrahedra (need for visibility sorting, scalability issues) [S1].
*   **Usage:** It is used for rendering large-scale irregular volume datasets, such as those from CFD and CSM simulations [S1].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
