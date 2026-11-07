# Uniform Sphere Tracing

Also called: Ray Marching for Volumes, Uniform Density Volume Rendering, Volume Rendering with Ray Marching (Uniform Sphere)

## Overview
Uniform Sphere Tracing renders participating media by advancing a ray through a volume using a constant step size derived from the scene's Signed Distance Function (SDF), while applying Beer's Law to compute transmission and scattering [S1, S2, S7, S14, S33, S48, unofficial]. It differs from adaptive methods by maintaining a fixed step size regardless of local density variations, prioritizing implementation simplicity and predictable memory access patterns over sampling efficiency [S1, S2, S7, S14, S33, S48, unofficial]. This method builds on [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md) for geometric traversal and applies volumetric shading models to the traced path.

## How it works
**Inputs and Outputs:**
- **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), maximum travel distance $t_{max}$, convergence threshold $\epsilon$, SDF function $f(\mathbf{p})$, volume absorption coefficient $\sigma_a$, volume scatter color $C_{vol}$, background color $C_{bg}$.
- **Output:** Final pixel color $C_{final}$.

**Data Structures:**
The scene geometry is represented by an SDF. For simple primitives like spheres, the structure includes:
- `center`: $\mathbb{R}^3$ vector defining the sphere center.
- `radius`: float defining the sphere radius.
- `sigma_a`: float defining the absorption coefficient.
- `scatter`: $\mathbb{R}^3$ vector defining the volume's intrinsic color.

**Algorithm Steps:**
1.  **Ray Initialization:** Compute the ray direction $\mathbf{d}$ and origin $\mathbf{o}$ for the current pixel. Initialize accumulated distance $t = 0$ and current position $\mathbf{p} = \mathbf{o}$.
2.  **Bounding Volume Check:** Query the SDF at $\mathbf{p}$. If $f(\mathbf{p}) > t_{max}$ or $f(\mathbf{p})$ indicates the ray is outside the bounding volume of interest, return $C_{bg}$ [S1, S2, S34, unofficial].
3.  **Sphere Tracing Loop:** While $t < t_{max}$ and $f(\mathbf{p}) > \epsilon$:
    a.  **Step Size:** Calculate step size $\Delta t = f(\mathbf{p})$ [S2, S7, S14, S33, unofficial].
    b.  **Advance:** Update position $\mathbf{p} \leftarrow \mathbf{p} + \Delta t \cdot \mathbf{d}$ and accumulated distance $t \leftarrow t + \Delta t$ [S14, S33, unofficial].
    c.  **Volumetric Integration:** At each step, compute the transmission factor $T_{step} = e^{-\sigma_a \cdot \Delta t}$ [S1, unofficial]. Accumulate the optical depth or integrate the color contribution. For a uniform volume, the total transmission $T$ through the entire path inside the volume is computed as $T = e^{-\sigma_a \cdot d_{total}}$, where $d_{total}$ is the total distance traveled within the volume [S1, unofficial].
4.  **Termination:**
    -   If $f(\mathbf{p}) \le \epsilon$, the ray has hit a surface. Compute the surface normal $\mathbf{n}$ using the gradient of the SDF: $\mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|}$ [S1, S10, unofficial].
    -   If $t \ge t_{max}$, the ray has escaped.
5.  **Color Computation:**
    -   If the ray hit a surface or escaped, calculate the final color. For a uniform volume sphere, if the ray intersects the sphere, calculate the entry point $\mathbf{p}_1$ and exit point $\mathbf{p}_2$ using intersection distances $t_0$ and $t_1$ [S1, unofficial].
    -   Calculate the distance through the volume $d = t_1 - t_0$ (assuming normalized $\mathbf{d}$) [S1, unofficial].
    -   Compute transmission $T = e^{-d \cdot \sigma_a}$ [S1, unofficial].
    -   Compute final color: $C_{final} = T \cdot C_{bg} + (1 - T) \cdot C_{vol}$ [S1, unofficial].
    -   If no intersection with the volume was found, return $C_{bg}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniform_sphere_tracing(ray_origin, ray_direction, sphere, t_max, epsilon):
    # ray_direction must be normalized
    # sphere: object with center, radius, sigma_a, scatter
    
    # 1. Check intersection with sphere
    t0, t1 = sphere_intersect(ray_origin, ray_direction, sphere.center, sphere.radius)
    
    if t0 is None:
        return sphere.background_color
        
    # 2. Calculate path length through volume
    # t0 is entry, t1 is exit. Ensure we are within t_max
    if t0 > t_max:
        return sphere.background_color
        
    t_exit = min(t1, t_max)
    distance = t_exit - t0
    
    # 3. Apply Beer's Law for transmission
    # T = exp(-sigma_a * distance)
    transmission = exp(-sphere.sigma_a * distance)
    
    # 4. Blend background and volume color
    # C_final = T * C_bg + (1 - T) * C_vol
    background_through_volume = transmission * sphere.background_color
    volume_scatter = (1.0 - transmission) * sphere.scatter
    
    return background_through_volume + volume_scatter

def sphere_intersect(origin, direction, center, radius):
    # Standard ray-sphere intersection
    oc = origin - center
    a = dot(direction, direction)
    b = 2.0 * dot(oc, direction)
    c = dot(oc, oc) - radius * radius
    discriminant = b * b - 4 * a * c
    
    if discriminant < 0:
        return None, None
        
    sqrt_disc = sqrt(discriminant)
    t0 = (-b - sqrt_disc) / (2.0 * a)
    t1 = (-b + sqrt_disc) / (2.0 * a)
    
    return t0, t1
```

**Parameters:**
-   $\sigma_a$: Absorption coefficient. Controls opacity. Typical value: $0.1$ [S1, unofficial].
-   $C_{vol}$: Scatter color. Typical value: $(0.8, 0.1, 0.5)$ [S1, unofficial].
-   $C_{bg}$: Background color. Typical value: $(0.572, 0.772, 0.921)$ [S1, unofficial].
-   $t_{max}$: Maximum travel distance. Not given in the sources.
-   $\epsilon$: Convergence threshold. Not given in the sources.

**Edge Cases and Pitfalls:**
-   **Normalized Direction:** The distance calculation $d = t_1 - t_0$ is valid only if the ray direction $\mathbf{d}$ is normalized. If not, $d = \|\mathbf{p}_2 - \mathbf{p}_1\|$ must be used [S1, unofficial].
-   **Precision:** Floating-point precision errors may cause the ray to slightly penetrate the surface or miss it entirely near the boundary. Using a small $\epsilon$ helps mitigate this [S1, S2, S14, S34, S39, unofficial].
-   **Uniform Assumption:** This method assumes uniform density within the volume. For non-uniform volumes, the integration step must be performed per step, not just at the end [S1, unofficial].

## Subtopics

### [Spectral Tracking](4664-spectral-tracking.md)
A variance reduction technique that handles per-wavelength optical properties by decomposing scattering and absorption coefficients into spectral components, extending the majorant-based sampling of Uniform Sphere Tracing.

### [Zeno Implementation](1544-zeno-implementation.md)
A specific implementation framework for sphere tracing methods.

### [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md)
A variant that supports gradient computation for optimization purposes.

### [Raymarched SDF Volume Rendering](3663-raymarched-sdf-volume-rendering.md)
A broader category of volume rendering using raymarching and SDFs.

### [Sphere Tracing with Composite SDF and Adaptive Sampling](3867-sphere-tracing-with-composite-sdf-and-adaptive-sampling.md)
An adaptive variant that adjusts step size based on local density.

### [Menger Sponge Raymarching](4875-menger-sponge-raymarching.md)
A specific application of sphere tracing to fractal geometry.

### [Sphere Tracing with Represent Hit](6684-sphere-tracing-with-represent-hit.md)
A variant focusing on hit representation.

## Operating characteristics
-   **Time Complexity:** $O(N)$, where $N$ is the number of steps. $N$ is determined by $t_{max} / \Delta t$. Since $\Delta t$ is constant, $N$ is proportional to the path length [S1, S2, S7, S14, S33, S48, unofficial].
-   **Memory Complexity:** $O(1)$ per ray, as only the current position and accumulated color/transmission need to be stored [S1, S2, S7, S14, S33, S48, unofficial].
-   **Performance:** Fast for simple, uniform volumes. Performance degrades for complex, non-uniform volumes requiring many small steps to maintain accuracy [S1, unofficial].
-   **Strengths:** Simple to implement, predictable memory access, no need for complex data structures like grids or trees for density [S1, S2, S7, S14, S33, S48, unofficial].
-   **Limitations:** Inefficient for sparse or highly variable density volumes, as it samples empty space or over-samples dense regions [S1, unofficial].
-   **When to Use:** Simple, uniform volumes, educational purposes, or when implementation simplicity is prioritized [S1, unofficial].
-   **When Not to Use:** Complex, non-uniform volumes, or when high performance is required for large scenes [S1, unofficial].

## Background
Uniform Sphere Tracing for volume rendering is a fundamental technique in computer graphics, often introduced in tutorials on ray marching and signed distance functions [S1, S2, S7, S14, S33, S48, unofficial]. It combines the geometric traversal capabilities of sphere tracing with the physical shading model of Beer's Law [S1, S2, S7, S14, S33, S48, unofficial]. The method is widely used in real-time graphics for simple volumetric effects like fog, smoke, and clouds [S1, unofficial].

## Sources
[S1] 体渲染简明教程 - BimAnt, https://bimant.com/blog/volume-rendering-brief-intro/, fetched 2026-10-30 (unofficial)
