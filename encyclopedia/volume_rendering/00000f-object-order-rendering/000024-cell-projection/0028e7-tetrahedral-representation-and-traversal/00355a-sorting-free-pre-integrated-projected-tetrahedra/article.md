# Sorting-free Pre-integrated Projected Tetrahedra

Also called: Pre-integrated Projected Tetrahedra (PBVR), Sorting-free approach that controls the fragment rendering by using the opacity value, Sorting-free technique for volume rendering of an irregular volume dataset

## Overview
Sorting-free Pre-integrated Projected Tetrahedra (SPT) is a volume rendering technique for irregular datasets that eliminates the need for global visibility sorting by treating the rendering process as a stochastic sampling event [S1]. Unlike conventional projected tetrahedra methods that require back-to-front compositing [Parent](26-cell-projection.md), SPT renders the volume multiple times, discarding fragments based on their opacity values to approximate the integrated brightness without alpha blending [S1]. It addresses the scalability issues of sorting-based methods and the low image quality (visible footprints) of particle-based methods like PBVR [S1].

## How it works
**Inputs:**
*   Irregular volume dataset decomposed into tetrahedral cells.
*   Transfer function mapping scalar values to density ($\rho$) and luminosity ($c$).
*   Camera parameters (viewing ray definition).
*   Repetition level $N$ (number of stochastic passes).

**Outputs:**
*   Final image brightness $B_{final}$, computed as the ensemble average of $N$ rendering passes.

**Data Structures:**
*   **Tetrahedral Mesh:** List of tetrahedra, each defined by 4 vertices $(x,y,z)$ and associated scalar values.
*   **Pre-integration Texture:** A 2D texture storing the integrated opacity $\alpha$ as a function of segment length $l$ and average differential opacity $\tau_{avg}$ (or density $\rho$), as defined in the [Cell projection](26-cell-projection.md) principles. This allows hardware-accelerated lookup of opacity for a given ray segment length [S1][Parent](26-cell-projection.md).

**Mathematical Model:**
The method uses the **density emitter model**, assuming particles follow a Poisson distribution [S1].
*   $\rho$: Particle density (number of particles per unit volume).
*   $\Delta t$: Length of a ray segment within a tetrahedron.
*   $\pi r^2 \rho \Delta t$: Expected number of particles in the segment (where $r$ is the effective particle radius, often normalized or absorbed into $\rho$ in the opacity formula).
*   Opacity $\alpha$ for a segment of length $\Delta t$:
    $$ \alpha = 1 - e^{-\rho \cdot \text{area} \cdot \Delta t} $$
    In the context of the pre-integrated texture lookup, this is typically simplified to $\alpha = 1 - e^{-\tau_{avg} l}$, where $l$ is the segment length and $\tau_{avg}$ is the average extinction coefficient [S1][Parent](26-cell-projection.md).
*   Brightness $B_k$ for a segment $k$:
    $$ B_k = c_k \cdot (1 - e^{-\rho \cdot \text{area} \cdot \Delta t}) $$
    where $c_k$ is the particle luminosity [S1].

**Algorithm Steps:**
1.  **Pre-computation:** Generate the pre-integration texture mapping $(\tau_{avg}, l) \rightarrow \alpha$ for the current transfer function. This avoids per-pixel exponential calculations [S1][Parent](26-cell-projection.md).
2.  **Projection:** Project each tetrahedron's vertices to screen space. Subdivide the projected cell into polygons based on the front and back faces [Parent](26-cell-projection.md).
3.  **Stochastic Rendering Loop (Repeated $N$ times):**
    a.  **Rasterization:** Rasterize the projected tetrahedra. For each fragment, determine the ray segment length $l$ and the average density $\rho$ (or $\tau_{avg}$) within the tetrahedron.
    b.  **Opacity Lookup:** Sample the pre-integration texture using $(\rho, l)$ to obtain the opacity $\alpha$ [S1][Parent](26-cell-projection.md).
    c.  **Stochastic Discard:** Generate a uniform random number $r \in [0, 1]$.
        *   If $r > \alpha$: **Discard** the fragment.
        *   If $r \le \alpha$: **Keep** the fragment.
    d.  **Color Assignment:** If kept, assign the fragment's color based on the luminosity $c$ (or emissive color) associated with the density $\rho$. If the fragment is discarded, it contributes nothing (transparent/black) to the accumulation buffer for this pass.
    e.  **Accumulation:** Add the resulting fragment color (or 0 if discarded) to an accumulation buffer.
4.  **Finalization:** After $N$ repetitions, divide the accumulation buffer by $N$ to obtain the final image brightness $B_{final}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_sorting_free(tetrahedra, transfer_function, camera, N):
    """
    Render an irregular volume using Sorting-free Pre-integrated Projected Tetrahedra.
    
    Args:
        tetrahedra: List of tetrahedra, each with vertices and scalar values.
        transfer_function: Function mapping scalar value -> (density rho, luminosity c).
        camera: Camera object with projection matrix and ray generation logic.
        N: Number of stochastic repetitions.
        
    Returns:
        final_image: Accumulated and averaged pixel brightness.
    """
    # Pre-compute pre-integration texture for opacity lookup
    # Maps (rho, length) -> alpha = 1 - exp(-rho * length)
    pre_int_texture = create_pre_integration_texture()
    
    # Initialize accumulation buffer
    accumulation_buffer = zeros_like(image_resolution)
    
    for n in range(N):
        # Clear frame buffer for this pass
        frame_buffer = zeros_like(image_resolution)
        
        # Project and rasterize each tetrahedron
        for tet in tetrahedra:
            # 1. Project vertices to screen space
            screen_vertices = project_vertices(tet.vertices, camera.projection_matrix)
            
            # 2. Subdivide projected cell into polygons (as per Cell Projection principles)
            polygons = subdivide_projected_cell(screen_vertices)
            
            # 3. Rasterize polygons
            for poly in polygons:
                # Interpolate segment length l and average density rho across the polygon
                # l varies linearly for orthogonal, approximated for perspective [Parent]
                for fragment in rasterize_polygon(poly):
                    l = interpolate_segment_length(fragment, tet)
                    rho = interpolate_density(fragment, tet, transfer_function)
                    
                    # Lookup pre-integrated opacity
                    alpha = pre_int_texture.sample(rho, l)
                    
                    # Stochastic test
                    random_val = random_uniform(0.0, 1.0)
                    
                    if random_val <= alpha:
                        # Fragment survives; get luminosity/color
                        c = transfer_function.get_luminosity(rho)
                        frame_buffer[fragment.coords] += c
                    else:
                        # Fragment discarded (contributes 0)
                        pass
        
        # Accumulate result from this pass
        accumulation_buffer += frame_buffer
        
    # Average over N repetitions
    final_image = accumulation_buffer / N
    return final_image
```

**Parameters:**
*   **Repetition Level ($N$):** Controls the variance of the stochastic result. Higher $N$ yields lower noise but increases computation time linearly. Typical values range from 1 (noisy, fast) to 100+ (smooth, slow) [S1].
*   **Pre-integration Texture Resolution:** Determines the accuracy of the opacity lookup. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Noise:** Low $N$ results in significant stochastic noise, especially in low-density regions [S1].
*   **Performance Scaling:** Performance is heavily influenced by the number of pixels. Memory consumption increases with image resolution because the number of particles (or stochastic events) scales with pixel count [S1].
*   **Precipitous Transfer Functions:** SPT handles sharp changes in density better than PBVR, which may generate insufficient particles in such regions [S1].
*   **Concave Sub-volumes:** Unlike sorting-based methods, SPT does not fail on concave sub-volumes because it does not rely on visibility ordering [S1].

## Operating characteristics
*   **Time Complexity:** The basic brightness calculation without stochastic approximation is $O(n^2)$ if intermediate brightness values are considered [S1]. With stochastic approximation, the complexity is $O(N \cdot P \cdot C)$, where $N$ is repetitions, $P$ is pixels, and $C$ is cells/tetrahedra. Performance is greatly influenced by the number of pixels [S1].
*   **Memory Complexity:** Memory usage is dominated by the accumulation buffer and the pre-integration texture. It is memory-consuming as image resolution increases [S1]. Unlike HAVS, it does not require extra memory for sorting cell faces (which is ~2x the size of tetrahedral cells) [S1].
*   **Measured Performance:** On an NVIDIA GeForce 9600GT, ~10 million particles (equivalent tetrahedral cells) were rendered at 25.7 milliseconds per repetition with 1024x1024 resolution [S1].
*   **Strengths:**
    *   No visibility sorting required, enabling parallel/distributed rendering of large-scale irregular volumes [S1].
    *   Handles concave sub-volumes correctly [S1].
    *   Avoids artifacts from insufficient k-buffer sizes (unlike HAVS) [S1].
    *   Better handling of precipitous transfer functions than PBVR [S1].
*   **Limitations:**
    *   Stochastic noise requires high $N$ for high-quality images, increasing render time [S1].
    *   Performance degrades with high image resolutions due to pixel count scaling [S1].
    *   Not suitable for real-time applications requiring high resolution and low noise without significant hardware acceleration [S1].
*   **When to Use:** Large-scale irregular volumes where visibility sorting is impractical or impossible (e.g., distributed computing environments), and high-quality real-time rendering is not strictly required [S1].
*   **When Not to Use:** When real-time performance at high resolutions is critical, or when the dataset is small enough that sorting-based methods (like HAVS) are faster and artifact-free [S1].

## Background
*   **Origin:** Proposed in the paper "Sorting-free pre-integrated projected tetrahedra" [S1].
*   **History:** Developed to address the limitations of particle-based volume rendering (PBVR) which suffers from visible footprints, and conventional projected tetrahedra which require complex visibility sorting. It extends the density emitter model and pre-integration techniques to a stochastic framework [S1].
*   **Usage:** Applied to large-scale irregular volumes from CSM (Computational Solid Mechanics) and CFD (Computational Fluid Dynamics) simulations [S1].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
