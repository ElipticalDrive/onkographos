# Sorting-free Pre-integrated Projected Tetrahedra

Also called: Pre-integrated Projected Tetrahedra, Sorting-free approach that controls the fragment rendering by using the opacity value, Sorting-free technique for volume rendering of an irregular volume dataset

## Overview
Sorting-free Pre-integrated Projected Tetrahedra (PBVR) is a volume rendering technique for irregular datasets that eliminates the need for explicit back-to-front visibility sorting [S1]. It solves the scalability bottlenecks of conventional projected tetrahedra and the image-quality artifacts of particle-based methods by treating rendering as a stochastic process where fragments are rendered with a probability equal to their opacity [S1]. Unlike [Projected Tetrahedra](877-projected-tetrahedra.md), which requires sorting, and [Particle-based Volume Rendering](877-projected-tetrahedra.md) (PBVR in the context of the source's comparison, though the method name is PBVR), which suffers from visible footprints, PBVR uses pre-integrated projected tetrahedra to achieve semi-transparency without alpha-blending or sorting [S1].

## How it works
**Inputs:** Irregular volume dataset decomposed into tetrahedra; transfer function mapping scalar values to density ($\rho$) and luminosity ($c$); camera parameters.
**Outputs:** Rendered image (brightness/color per pixel).

**Data Structures:**
The method operates on tetrahedral cells. Each cell is defined by 4 vertices. The scalar field is interpolated linearly within the tetrahedron [S1]. The core data structure for the stochastic process is the set of ray segments intersecting the tetrahedra, characterized by constant density $\rho_k$ and luminosity $c_k$ per segment [S1].

**Formulas:**
1.  **Opacity:** The opacity $\tau$ for a ray segment of length $\Delta t$ with particle density $\rho$ (particles per unit volume) and effective cross-sectional area $\pi r^2$ is derived from the Poisson distribution assumption:
    $$ \tau = 1 - e^{-\rho \cdot \pi r^2 \cdot \Delta t} $$
    [S1]. Note: In the context of projected tetrahedra, $\rho$ represents the scalar value mapped to density, and the geometric intersection length $\Delta t$ is used. The source text uses $\pi r^2 \rho \Delta t$ to express the expected number of particles in the segment [S1].

2.  **Brightness (Deterministic):** The brightness $B_k$ for a segment $k$ with constant luminosity $c_k$ and opacity $\tau_k$ is:
    $$ B_k = c_k \times (1 - e^{-\rho \cdot \text{area} \cdot \Delta t}) $$
    [S1].

3.  **Stochastic Probability:** The probability that the $k$-th segment contributes its luminosity (i.e., "hits") is defined by the event where no particles are detected in previous segments and at least one is detected in the current segment [S1]. However, the implementation simplifies this to a direct sampling based on opacity.

**Algorithm Steps:**
1.  **Decomposition:** Subdivide the viewing ray into segments where particle luminosity $c_k$ and density $\rho_k$ are constant [S1].
2.  **Opacity Calculation:** Calculate the opacity $\tau_k$ for each segment using the density emitter model and Poisson distribution assumption [S1].
3.  **Stochastic Rendering (Repetition):**
    *   Initialize the accumulated brightness for each pixel to zero.
    *   Repeat the following $N$ times (where $N$ is the repetition level):
        1.  **Rasterization:** Rasterize the projected tetrahedra. For each fragment generated, evaluate the opacity $\tau$ at the fragment's position [S1].
        2.  **Probabilistic Discard:** Generate a random number $r \in [0, 1]$. If $r > \tau$, discard the fragment (do not write to the buffer). If $r \le \tau$, render the fragment with its luminosity/color [S1].
        3.  **Accumulation:** Add the rendered fragment's value to the pixel's accumulated brightness [S1].
    *   **Averaging:** Divide the accumulated brightness by $N$ to obtain the final pixel value [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pbvr(tetrahedra, transfer_function, camera, width, height, repetition_level):
    """
    Renders an irregular volume using sorting-free pre-integrated projected tetrahedra.
    
    Args:
        tetrahedra: List of tetrahedra, each with 4 vertices and associated scalar values.
        transfer_function: Function mapping scalar value s to density rho and luminosity c.
        camera: Camera parameters for projection.
        width, height: Image dimensions.
        repetition_level: Number of stochastic repetitions N.
        
    Returns:
        Image buffer of size (height, width, 3) or (height, width, 4).
    """
    # Initialize accumulation buffer
    # Using float32 for precision during accumulation
    acc_buffer = np.zeros((height, width, 3), dtype=np.float32)
    
    for rep in range(repetition_level):
        # Create a temporary buffer for this repetition
        # Initialize with 0 (transparent/black)
        rep_buffer = np.zeros((height, width, 3), dtype=np.float32)
        
        # Rasterize each tetrahedron
        for tet in tetrahedra:
            # Project tetrahedron vertices to screen space
            screen_vertices = project_to_screen(tet.vertices, camera)
            
            # Determine bounding box for rasterization
            min_x, max_x, min_y, max_y = get_bounding_box(screen_vertices)
            min_x, max_x = clamp_to_image(min_x, max_x, width)
            min_y, max_y = clamp_to_image(min_y, max_y, height)
            
            # Rasterize the triangle faces of the tetrahedron
            # A tetrahedron has 4 triangular faces
            for face in tet.faces:
                face_vertices = [screen_vertices[v] for v in face]
                
                # Iterate over pixels in the bounding box
                for y in range(int(min_y), int(max_y) + 1):
                    for x in range(int(min_x), int(max_x) + 1):
                        # Check if pixel (x, y) is inside the projected triangle
                        if is_inside_triangle(x, y, face_vertices):
                            # Interpolate scalar value at pixel center in 3D
                            # This requires back-projecting to 3D or using barycentric coords
                            scalar_value = interpolate_scalar_at_pixel(tet, x, y, camera)
                            
                            # Map scalar to density and luminosity
                            rho, c = transfer_function(scalar_value)
                            
                            # Calculate opacity from density
                            # Note: The exact geometric length delta_t and area factor 
                            # are handled by the pre-integration or rasterization context.
                            # Here we assume the opacity tau is pre-computed or derived 
                            # from the scalar value via the transfer function's density mapping.
                            # In the source, tau = 1 - exp(-rho * area * delta_t).
                            # For projected tetrahedra, the "area * delta_t" is effectively 
                            # handled by the projection and the density value.
                            # We assume 'tau' is the final opacity value for the fragment.
                            tau = compute_opacity(rho) 
                            
                            # Stochastic sampling
                            r = random_float(0.0, 1.0)
                            if r <= tau:
                                # Fragment is "hit"
                                # Add luminosity to the buffer
                                rep_buffer[y, x] += c
                            else:
                                # Fragment is discarded (remains 0 or previous value if not cleared)
                                pass
        
        # Accumulate the result of this repetition
        acc_buffer += rep_buffer
        
    # Average over repetitions
    final_image = acc_buffer / repetition_level
    
    return final_image

def compute_opacity(rho):
    """
    Computes opacity from density.
    In the source, opacity is derived from the Poisson distribution assumption.
    tau = 1 - exp(-rho * area * delta_t).
    For the purpose of the stochastic fragment shader, the opacity value 
    passed to the random test is the probability of the fragment being opaque.
    """
    # The source implies that the opacity value controls the fragment rendering.
    # We return the probability tau.
    # Note: The exact scaling factor (area * delta_t) is absorbed into the 
    # transfer function or pre-integration lookup in practical implementations.
    # Here we assume rho is scaled appropriately.
    return 1.0 - np.exp(-rho)

def interpolate_scalar_at_pixel(tet, x, y, camera):
    """
    Interpolates the scalar value at the 3D point corresponding to pixel (x,y).
    This involves ray-tetrahedron intersection and barycentric interpolation.
    """
    # 1. Unproject pixel (x,y) to a ray in 3D space
    ray_origin, ray_direction = unproject_pixel(x, y, camera)
    
    # 2. Intersect ray with tetrahedron
    t_near, t_far = intersect_ray_tetrahedron(ray_origin, ray_direction, tet.vertices)
    
    if t_near is None:
        return None # No intersection
        
    # 3. Calculate intersection point
    point = ray_origin + ray_direction * t_near
    
    # 4. Compute barycentric coordinates of point within tetrahedron
    bary = compute_barycentric_coords(point, tet.vertices)
    
    # 5. Interpolate scalar values from vertices
    scalar = sum(bary[i] * tet.vertex_scalars[i] for i in range(4))
    
    return scalar
```

**Parameters:**
*   **Repetition Level ($N$):** The number of times the rendering process is repeated. Higher $N$ reduces noise/error but increases computation time. Typical values tested: 1, 25, 100, 144 [S1].
*   **Resolution:** Image resolution (e.g., 1024x1024) [S1].

**Edge Cases and Pitfalls:**
*   **Noise:** Low repetition levels result in noisy images. The error decreases with $N$ [S1].
*   **Precipitous Transfer Functions:** PBVR may generate an insufficient number of particles (or hits) when the density function changes drastically, leading to artifacts. [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md) (SPT) handles these better [S1].
*   **Memory:** Performance is greatly influenced by the number of pixels. Higher resolution increases the number of particles/hits to track, making it memory-consuming [S1].
*   **Concave Sub-volumes:** Conventional visibility sorting fails for concave sub-volumes; PBVR avoids this by not requiring sorting [S1].

## Operating characteristics
*   **Time Complexity:** The basic brightness calculation is $O(n^2)$ if intermediate brightness values are considered [S1]. The stochastic approach requires $N$ repetitions, so the total time is proportional to $N \times \text{Rasterization Time}$ [S1].
*   **Memory Complexity:** Memory usage is influenced by the number of pixels and the number of particles. It is memory-consuming as image resolution increases [S1].
*   **Performance:** On an NVIDIA GeForce 9600GT, ~10 million particles (tetrahedra) were rendered at 25.7 milliseconds per repetition with 1024x1024 resolution [S1].
*   **Strengths:** No visibility sorting required; scalable to large irregular volumes distributed across multiple nodes; avoids artifacts from insufficient k-buffer size (unlike HAVS); avoids visible footprints (unlike particle-based methods) [S1].
*   **Limitations:** Noisy at low repetition levels; sensitive to precipitous transfer functions; performance degrades with high resolution due to pixel count [S1].
*   **When to Use:** Large-scale irregular volumes where sorting is impractical or impossible (e.g., distributed computing); when high-quality iso-surface visualization is not critical compared to volume appearance [S1].
*   **When Not to Use:** When real-time performance with very high repetition levels is required; when transfer functions have sharp discontinuities [S1].

## Background
*   **Origin:** The method was introduced in the paper "Sorting-free pre-integrated projected tetrahedra" [S1].
*   **History:** It was developed to address the limitations of particle-based volume rendering (low image quality, visible footprints) and conventional projected tetrahedra (need for visibility sorting, scalability issues) [S1].
*   **Usage:** Applied to large-scale irregular volumes from CSM and CFD simulations [S1].

## Sources
[S1] Sorting-free pre-integrated projected tetrahedra, https://doi.org/10.1145/1838544.1838546, fetched 2026-11-01
