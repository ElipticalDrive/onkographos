# Sorting-free Pre-integrated Projected Tetrahedra

Also called: Pre-integrated Projected Tetrahedra, Sorting-free approach that controls the fragment rendering by using the opacity value, Sorting-free technique for volume rendering of an irregular volume dataset

## Overview
Sorting-free Pre-integrated Projected Tetrahedra (PBVR) is a volume rendering technique for irregular datasets that eliminates the need for explicit visibility sorting of tetrahedral cells [S1]. It solves the scalability issues of conventional sorting-based methods and the low image quality (visible footprints) of particle-based methods by treating volume rendering as a stochastic process [S1]. Unlike its sibling [Parallel Cell Projection](875-parallel-cell-projection.md) which may require sorting, PBVR controls fragment rendering probability based on opacity, allowing semi-transparent compositing without alpha-blending or back-to-front ordering [S1].

## How it works
PBVR builds on [Cell projection](26-cell-projection.md) by replacing deterministic compositing with a stochastic sampling approach applied to projected tetrahedra [S1].

### Inputs and Outputs
*   **Input:** Irregular volume dataset represented as a set of tetrahedral cells; camera parameters (viewing ray); transfer function mapping density to luminosity/opacity.
*   **Output:** Rendered image (brightness values per pixel).

### Data Structures
*   **Tetrahedral Mesh:** A collection of tetrahedra defined by vertex coordinates and scalar density values.
*   **Ray Segments:** Viewing rays are subdivided into segments where particle luminosity ($c_k$) and density ($\rho_k$) are treated as constant [S1].
*   **Repetition Buffer:** Since the method is stochastic, results are accumulated over $N$ repetitions to approximate the expected brightness [S1].

### Mathematical Model
The method uses the **density emitter model**, assuming the number of particles in a unit volume follows a Poisson distribution [S1].

1.  **Opacity Calculation:**
    The opacity $\tau$ for a ray segment of length $\Delta t$ is derived from the Poisson distribution assumption:
    $$ \tau = 1 - e^{-\rho \cdot \pi r^2 \cdot \Delta t} $$
    Where:
    *   $\rho$ is the number of particles in a unit volume (density).
    *   $r$ is the effective radius of the particle/cell interaction (often implicit in the pre-integration or cell geometry).
    *   $\Delta t$ is the ray-segment length [S1].
    *   Note: The source text uses $\pi r^2 \rho \Delta t$ to express the expected number of particles in the segment [S1].

2.  **Brightness Calculation (Deterministic Baseline):**
    The brightness $B_k$ for a segment $k$ with constant luminosity $c_k$ and density $\rho$ is:
    $$ B_k = c_k \times (1 - e^{-\rho \cdot \text{area} \cdot \text{length}}) $$
    Where "area" and "length" correspond to the geometric cross-section and ray segment length involved in the pre-integration [S1]. The computational complexity of calculating intermediate brightness values is $O(n^2)$ if not optimized [S1].

3.  **Stochastic Rendering (Sorting-Free Step):**
    To avoid sorting, the rendering is repeated $N$ times. In each repetition:
    *   Fragments are rendered with a probability equal to the calculated opacity value at each ray-segment along the viewing ray [S1].
    *   If a fragment is rendered, its luminosity contributes to the pixel brightness.
    *   If discarded, it contributes 0.
    *   The final image is the ensemble average (mean) of the brightness values from all $N$ repetitions [S1].

    The probability that the $k$-th luminosity event occurs (no particles in previous segments, $\ge 1$ particle in current segment) is defined by the Poisson distribution properties [S1]:
    $$ P(\text{event}_k) = e^{-\lambda_{k-1}} (1 - e^{-\lambda_k}) $$
    Where $\lambda$ represents the expected number of particles in the respective segments [S1].

### Algorithm Steps
1.  **Pre-computation:** Construct a 3D texture for pre-integrated classification based on the transfer function. This step takes approximately 100 milliseconds [S1].
2.  **Ray Subdivision:** For each pixel, subdivide the viewing ray into segments where density $\rho$ and luminosity $c$ are constant [S1].
3.  **Opacity Evaluation:** Calculate the opacity $\tau$ for each segment using the density emitter model formula [S1].
4.  **Stochastic Rendering Loop (Repeat $N$ times):**
    a.  Rasterize the projected tetrahedra.
    b.  For each fragment generated, generate a random number $r \in [0, 1]$.
    c.  Compare $r$ with the fragment's opacity $\tau$.
    d.  If $r > \tau$, discard the fragment (contribution 0).
    e.  If $r \le \tau$, render the fragment with its luminosity value $c$ [S1].
5.  **Accumulation:** Average the pixel values from all $N$ repetitions to obtain the final brightness [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pbvr(tetrahedra, camera, transfer_function, num_repetitions):
    """
    Renders an irregular volume using Sorting-free Pre-integrated Projected Tetrahedra.
    
    Args:
        tetrahedra: List of tetrahedral cells with density and geometry.
        camera: Camera object providing ray generation.
        transfer_function: Function mapping density to luminosity (c) and opacity parameters.
        num_repetitions: N, the number of stochastic samples per pixel.
        
    Returns:
        image: 2D array of final brightness values.
    """
    # Pre-compute pre-integrated classification texture (takes ~100ms)
    pre_integrated_texture = build_pre_integrated_texture(transfer_function)
    
    # Initialize accumulation buffer
    # Dimensions match image resolution (e.g., 1024x1024)
    accumulation_buffer = zeros(image_width, image_height)
    
    for repetition in range(num_repetitions):
        # Initialize frame buffer for this repetition
        frame_buffer = zeros(image_width, image_height)
        
        # Project and rasterize tetrahedra
        # This step leverages hardware rasterization of projected tetrahedra
        projected_faces = project_and_rasterize(tetrahedra, camera)
        
        for fragment in projected_faces:
            # Get density at fragment's position in volume space
            density = sample_density(fragment.position, tetrahedra)
            
            # Evaluate transfer function
            luminosity = transfer_function.get_luminosity(density)
            # Opacity tau derived from density emitter model
            # tau = 1 - exp(-rho * area * length)
            # In practice, this is often looked up or computed from pre-integrated data
            opacity = transfer_function.get_opacity(density)
            
            # Stochastic decision: render with probability = opacity
            random_val = random_float(0.0, 1.0)
            
            if random_val <= opacity:
                # Fragment is "active" for this repetition
                frame_buffer[fragment.screen_x][fragment.screen_y] += luminosity
            else:
                # Fragment is discarded (contributes 0)
                pass
        
        # Accumulate results
        accumulation_buffer += frame_buffer
        
    # Final average
    image = accumulation_buffer / num_repetitions
    return image
```

### Parameters and Typical Values
*   **Repetition Level ($N$):** Controls image quality. Higher $N$ reduces error. Typical values tested: 1, 25, 100, 144 [S1].
*   **Resolution:** Performance is heavily influenced by pixel count. Tested at 1024x1024 [S1].
*   **Pre-integration Texture:** Required for efficient opacity/luminosity lookup [S1].

### Edge Cases and Pitfalls
*   **Drastic Density Changes:** PBVR may generate an insufficient number of particles (or active fragments) when the density function changes drastically, leading to artifacts. Sorting-free Pre-integrated Projected Tetrahedra (SPT) handles precipitous transfer functions better than pure PBVR [S1].
*   **Memory Consumption:** PBVR is memory-consuming because increasing image resolution affects the number of particles/fragments processed [S1].
*   **Performance Bottleneck:** Performance is greatly influenced by the number of pixels. For large datasets (>24.5 million cells), PBVR is preferred over SPT, but for smaller datasets, SPT may be faster [S1].
*   **Complexity:** The basic brightness calculation has $O(n^2)$ complexity if intermediate brightness values are considered, though the stochastic approach avoids the sorting bottleneck [S1].

## Operating characteristics
*   **Time Complexity:** The stochastic rendering loop runs $N$ times. The cost per repetition depends on the number of projected tetrahedra and pixels. The base brightness calculation is $O(n^2)$ [S1].
*   **Memory Complexity:** Memory usage scales with image resolution and the number of particles. PBVR is more memory-consuming than sorting-based methods like HAVS because it does not require large k-buffers for sorting, but the stochastic nature requires storing accumulation buffers [S1]. HAVS requires memory space for sorting cell faces about twice that for tetrahedral cells [S1].
*   **Performance:** On an NVIDIA GeForce 9600GT, PBVR rendered ~10 million particles at 25.7 milliseconds per repetition with 1024x1024 resolution [S1]. For 1024x1024 resolution, the computing time is estimated at 1.4 seconds per repetition for large datasets [S1].
*   **Strengths:** No visibility sorting required; handles large-scale irregular volumes distributed across multiple nodes; avoids artifacts from insufficient k-buffer size (unlike HAVS); avoids visible footprints (unlike PBVR particle methods) [S1].
*   **Limitations:** Stochastic noise requires multiple repetitions for high quality; performance degrades with high resolution; less effective than SPT for precipitous transfer functions [S1].
*   **When to Use:** Large-scale irregular volumes where sorting is impractical or impossible (e.g., distributed computing); when visible footprints of particle methods are unacceptable [S1].
*   **When Not to Use:** Small datasets where sorting is fast; scenes with precipitous transfer functions where SPT is superior; real-time applications requiring very high frame rates without averaging [S1].

## Background
*   **Origin:** The method was introduced in the paper "Sorting-free pre-integrated projected tetrahedra" [S1].
*   **History:** It was developed to address the limitations of particle-based volume rendering (low image quality, visible footprints) and conventional projected tetrahedra (requirement for visibility sorting, which is difficult for large-scale irregular volumes) [S1]. It builds on the density emitter model and Poisson distribution assumptions for particle counts [S1].
*   **Usage:** Applied to large-scale CSM (Computational Solid Mechanics) and CFD (Computational Fluid Dynamics) simulation results [S1].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
