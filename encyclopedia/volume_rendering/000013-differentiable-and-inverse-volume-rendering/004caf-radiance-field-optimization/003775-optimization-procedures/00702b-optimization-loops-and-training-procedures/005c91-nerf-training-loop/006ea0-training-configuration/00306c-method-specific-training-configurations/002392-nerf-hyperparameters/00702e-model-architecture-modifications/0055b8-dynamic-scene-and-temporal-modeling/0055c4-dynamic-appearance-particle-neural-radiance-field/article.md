# Dynamic Appearance Particle Neural Radiance Field

Also called: Dynamic Appearance Particle NeRF, Lagrangian dynamic model in NeRF

## Overview
Dynamic Appearance Particle Neural Radiance Field (DAP-NeRF) is a method for novel view synthesis in dynamic scenes using monocular video input. It solves the problem of modeling temporal dynamics without requiring multi-view data or prior geometric knowledge of moving objects [S1]. Unlike ParticleNeRF, which encodes the entire scene with particles and requires multiple camera views per frame, DAP-NeRF applies appearance particles only to dynamic elements while maintaining a static grid-based field for the background, enabling efficient linear interpolation and monocular training [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Monocular video data (a sequence of images with known camera poses and timestamps $t$).
*   **Outputs:** Rendered pixel colors $C(\mathbf{r}, t)$ for arbitrary viewpoints and times.

**Data Structures:**
The method uses a hybrid representation defined by the superposition equation:
$$ V(\mathbf{x}, t) = (1 - w_t(\mathbf{x}))V_s(\mathbf{x}) + w_t(\mathbf{x})V_{td}(\mathbf{x}) $$
where:
*   $V_s(\mathbf{x})$: Static field component. Implemented as a time-independent grid-based model (e.g., canonical feature field) [S1].
*   $V_{td}(\mathbf{x})$: Dynamic field component. Implemented as an ensemble of appearance particles [S1].
*   $w_t(\mathbf{x})$: Weight function. Takes value 1 if the dynamic field is effective at location $\mathbf{x}$ and time $t$, and 0 otherwise [S1]. The support of the dynamic field is implied by the particle representation [S1].

**Notation Convention:**
Time $t$ is denoted in subscripts on the right-hand side of the superposition equation (e.g., $V_{td}$) to clarify that the dynamic component consists of an ensemble of particles, where the time argument is fixed for the instantaneous status construction [S1].

**Algorithm Steps:**

1.  **Ray Casting:** For a target pixel at time $t$, cast a ray $\mathbf{r}(s) = \mathbf{o} + s\mathbf{d}$ from camera center $\mathbf{o}$ with direction $\mathbf{d}$, where $s \in \mathbb{R}^+$ [S1].
2.  **Field Evaluation:** For each sample point $\mathbf{x} = \mathbf{r}(s)$ along the ray between near bound $s_n$ and far bound $s_f$:
    a. Query the static field $V_s(\mathbf{x})$ using the grid-based model [S1].
    b. Query the dynamic field $V_{td}(\mathbf{x})$. This is formulated as a convolution with a kernel $W$ with finite support, replacing the Dirac function $\delta$ to quantize the field:
       $$ V_{td}(\mathbf{x}) = (V_{td} * W)(\mathbf{x}) = \int V_{td}(\mathbf{x}')W(\mathbf{x} - \mathbf{x}')d\nu(\mathbf{x}') $$
       where $W(\mathbf{x}) = 0$ when $\|\mathbf{x}\|$ exceeds a small radius, and $\int W(\mathbf{x})d\nu(\mathbf{x}) = 1$ [S1].
    c. Compute the combined field value $V(\mathbf{x}, t)$ using the superposition formula [S1].
3.  **Rendering:** Integrate the transmittance $T(s)$ and color $C(\mathbf{r}, t)$ along the ray:
    $$ C(\mathbf{r}, t) = \int_{s_n}^{s_f} T(s)\sigma(\mathbf{r}(s), t)c(\mathbf{r}(s), \mathbf{d}, t)ds $$
    where $T(s) = \exp\left(-\int_{s_n}^{s} \sigma(\mathbf{r}(s'), t)ds'\right)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, time_t, static_grid, dynamic_particles, kernel_width):
    """
    Renders a single pixel for a given ray and time using DAP-NeRF.
    
    Args:
        ray_origin: 3D vector, camera center o.
        ray_direction: 3D vector, viewing direction d.
        time_t: float, timestamp t.
        static_grid: Grid structure storing V_s(x).
        dynamic_particles: List of particles, each with position, velocity, and appearance features.
        kernel_width: float, radius of the finite support kernel W.
        
    Returns:
        color: 3D vector, rendered pixel color C(r, t).
    """
    # 1. Sample points along the ray
    samples = sample_points_along_ray(ray_origin, ray_direction, near_bound=0.1, far_bound=10.0, num_samples=128)
    
    accumulated_transmittance = 1.0
    accumulated_color = [0.0, 0.0, 0.0]
    
    for point in samples:
        # 2. Evaluate Static Field V_s(x)
        # Query the time-independent grid-based model
        static_value = static_grid.query(point)
        
        # 3. Evaluate Dynamic Field V_td(x)
        # Compute convolution with kernel W over dynamic particles
        dynamic_value = 0.0
        for particle in dynamic_particles:
            # Calculate distance from particle to sample point
            dist = norm(particle.position - point)
            
            # Apply kernel W (finite support)
            # W(x) = 0 if ||x|| > kernel_width, normalized otherwise
            if dist <= kernel_width:
                weight = kernel_function(dist, kernel_width)
                # Interpolate particle features (e.g., density, color)
                particle_features = interpolate_particle_features(particle, point)
                dynamic_value += weight * particle_features
        
        # 4. Superposition
        # Determine weight w_t(x). 
        # Note: In practice, w_t(x) is 1 if dynamic field is effective, 0 otherwise.
        # Effectiveness is implied by particle support. 
        # If dynamic_value is negligible (outside particle support), w_t = 0.
        # Otherwise, w_t = 1.
        if dynamic_value > epsilon:
            w_t = 1.0
        else:
            w_t = 0.0
            
        combined_value = (1 - w_t) * static_value + w_t * dynamic_value
        
        # Extract density sigma and color c from combined_value
        sigma = extract_density(combined_value)
        color = extract_color(combined_value)
        
        # 5. Volume Rendering Integration
        # T(s) = exp(-int(sigma ds))
        # Contribution = T(s) * (1 - exp(-sigma * delta_s)) * color
        # Simplified for discrete sampling:
        alpha = 1.0 - exp(-sigma * delta_s)
        transmittance = accumulated_transmittance * (1.0 - alpha)
        
        accumulated_color += transmittance * color
        accumulated_transmittance *= (1.0 - alpha)
        
        # Early termination if transmittance is very low
        if accumulated_transmittance < epsilon:
            break
            
    return accumulated_color

def kernel_function(dist, radius):
    """
    Finite support kernel W(x).
    W(x) = 0 when ||x|| exceeds radius.
    Normalized such that integral is 1.
    """
    if dist > radius:
        return 0.0
    # Simple box kernel or Gaussian truncated at radius
    # Exact normalization factor depends on dimensionality and kernel shape.
    # Not given in sources. [G]
    return 1.0 / (4/3 * pi * radius**3) # Assuming 3D box kernel normalization
```

**Parameters:**
*   $w_t(\mathbf{x})$: Binary weight (0 or 1) determining the effectiveness of the dynamic field at $(\mathbf{x}, t)$ [S1].
*   Kernel support radius: Small radius defining the finite support of $W$ [S1].
*   Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The transition between static and dynamic fields relies on the binary weight $w_t(\mathbf{x})$. Discontinuities may occur at the boundary of particle support if not smoothed.
*   **Boundaries:** Ray bounds $s_n$ and $s_f$ must be set appropriately to capture the scene extent [S1].
*   **Degenerate Input:** The method assumes monocular video. If multi-view data is available, the method still functions but does not leverage it for dynamic reconstruction in the same way ParticleNeRF does [S1].

## Subtopics

### ParticleNeRF
Encodes the entire scene with particles and uses radius nearest neighbor searching. Requires 20 camera views per time frame. DAP-NeRF differs by using particles only for dynamic elements and supporting monocular input [S1].

### InstantNGP
A fast training framework. DAP-NeRF is not tightly coupled to a specific architecture and can be integrated into InstantNGP [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Handles monocular video data (one camera view per time frame); does not require prior geometric knowledge of dynamic objects; efficient querying via linear interpolation on a grid-based field for static parts [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When only monocular video is available and dynamic objects need to be modeled without explicit mesh extraction.
*   **When not:** When multi-view data is available and full scene particle encoding is preferred (e.g., ParticleNeRF) [S1].

## Background
The method was proposed in "Dynamic Appearance Particle Neural Radiance Field" [S1]. It builds on the Lagrangian dynamic model, embedding it into the radiance field representation. It addresses limitations of concurrent works that rely on specific physical models or pre-extracted shapes [S1]. It differs from ParticleNeRF [S1] by focusing on dynamic elements only and supporting monocular input.

## Sources
[S1] Dynamic Appearance Particle Neural Radiance Field, https://doi.org/10.1109/tcsvt.2025.3540792, fetched 2026-11-05
