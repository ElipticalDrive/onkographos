# Ray Casting and Compositing Loop

Also called: Front-to-back compositing loop, Volume rendering integral evaluation

## Overview
This method defines the core computational loop for direct volume ray casting, responsible for sampling the volume, mapping data to optical properties, and accumulating color and opacity along a ray [S5, S10, S37]. It solves the problem of synthesizing a 2D image from a 3D scalar or vector field by evaluating the volume rendering equation [S1, S13]. It is distinguished from [Ray Marching and Discretization](2788-ray-marching-and-discretization.md) by its explicit handling of the compositing accumulation and transfer function application within the integration step, and from [Monte Carlo Path Tracing for Volume Rendering](4980-monte-carlo-path-tracing-for-volume-rendering.md) by its deterministic, image-order approach [S1, S56].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, direction $\mathbf{d}$, volume data structure $V$, transfer function $T$, background color $C_{bg}$.
**Outputs:** Final pixel color $C_{final}$ and opacity $\alpha_{final}$.

**Data Structures:**
*   **Ray:** Parameterized as $r(t) = \mathbf{o} + t\mathbf{d}$.
*   **Volume:** Accessible via interpolation. For regular grids, this is typically a 3D array; for unstructured grids (e.g., MLS), it requires neighbor queries [S1, unofficial].
*   **Accumulators:** $C_{acc}$ (color, initialized to 0), $\alpha_{acc}$ (opacity, initialized to 0).

**Steps:**

1.  **Ray-Volume Intersection:** Determine the entry $t_{start}$ and exit $t_{end}$ points where the ray intersects the volume boundary [S1, S8, S16]. For non-convex volumes, this may yield multiple intervals [S1, unofficial].
2.  **Discretization:** Divide the segment $[t_{start}, t_{end}]$ into $N$ samples. The step size is $\Delta t = (t_{end} - t_{start}) / N$ [S5, S10, S17]. Adaptive schemes may vary $\Delta t$ based on local frequency or error estimates [S1, unofficial].
3.  **Sampling Loop:** For each sample $i$ from front to back:
    a.  **Position:** Compute sample point $\mathbf{p}_i = r(t_i)$.
    b.  **Interpolation:** Retrieve scalar value $d_i$ from $V$ at $\mathbf{p}_i$ using a reconstruction kernel (e.g., trilinear interpolation) [S7, S10, S43].
    c.  **Classification:** Map $d_i$ to color $C_i$ and opacity $\alpha_i$ using the transfer function $T(d_i) \rightarrow (C_i, \alpha_i)$ [S2, S2789].
    d.  **Step Size Correction:** Adjust opacity for the discrete step size to approximate continuous integration: $\alpha'_i = 1 - (1 - \alpha_i)^{\Delta t}$ [S11, S32].
    e.  **Compositing:** Update accumulators:
        $$
        C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha'_i C_i
        $$
        $$
        \alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha'_i
        $$
        where $(1 - \alpha_{acc}^{(i-1)})$ is the transmittance [S1, S51].
    f.  **Early Termination:** If $\alpha_{acc}^{(i)} > \tau$ (threshold, typically 0.99), break the loop [S1, S10, S12].
4.  **Background Compositing:** Compute final color: $C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}$ [S1, S14].

**Pitfalls:**
*   **Boundary Intersection:** For methods relying on local approximations (like MLS), intersection points are critical because the approximation is ill-defined away from data points [S1, unofficial].
*   **Precision:** Accumulating small opacity values can lead to precision issues; using the transmittance form $T_i = T_{i-1} \cdot (1 - \alpha'_i)$ is numerically more stable for highly transparent volumes [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_casting_loop(ray_origin, ray_direction, volume, transfer_function, background_color, threshold=0.99):
    # 1. Intersection
    t_start, t_end = intersect_volume(ray_origin, ray_direction, volume)
    if t_start is None:
        return background_color, 0.0

    # 2. Discretization
    num_samples = estimate_samples(t_end - t_start, volume)
    delta_t = (t_end - t_start) / num_samples
    
    acc_color = [0.0, 0.0, 0.0]
    acc_alpha = 0.0
    transmittance = 1.0

    # 3. Sampling Loop
    for i in range(num_samples):
        t = t_start + i * delta_t
        point = ray_origin + ray_direction * t
        
        # Check if point is inside volume (for non-convex or adaptive steps)
        if not is_inside_volume(point, volume):
            continue

        # Sampling and Interpolation
        scalar_val = volume.interpolate(point)
        
        # Classification
        color, alpha = transfer_function(scalar_val)
        
        # Step Size Correction
        alpha_corrected = 1.0 - (1.0 - alpha) ** delta_t
        
        # Compositing
        contribution = transmittance * alpha_corrected * color
        acc_color = [c + contrib for c, contrib in zip(acc_color, contribution)]
        acc_alpha += contribution
        transmittance *= (1.0 - alpha_corrected)
        
        # Early Termination
        if acc_alpha > threshold:
            break

    # 4. Background Compositing
    final_color = [c + (1.0 - acc_alpha) * bg for c, bg in zip(acc_color, background_color)]
    return final_color, acc_alpha
```

## Subtopics

### CUDA GPU Implementation
A parallel implementation where each thread processes a single ray, decoupling traversal from the data structure [S1, unofficial]. It uses a dynamic redundant octree for neighbor queries and manages adaptive recursion via a local stack [S1, unofficial].

## Operating characteristics
*   **Complexity:** $O(N \cdot P)$ where $N$ is the number of pixels and $P$ is the number of samples per ray. Adaptive schemes reduce $P$ in smooth regions but increase overhead for traversal logic [S1, unofficial].
*   **Performance:** On CPU, interactive rates (e.g., 10-20 fps) are achievable for moderate resolutions with constant step sizes [S1, unofficial]. GPU implementations achieve higher throughput by parallelizing rays [S1, unofficial].
*   **Strengths:** Simple to implement, produces high-quality results with proper transfer functions, supports early termination [S10].
*   **Limitations:** Performance degrades with high opacity or complex transfer functions requiring many samples; adaptive schemes add complexity [S1, unofficial].

## Background
The front-to-back compositing loop is a fundamental technique in direct volume rendering, derived from the volume rendering integral [S1, S13]. It has been standard since the early days of DVR [S5, S10]. The specific use of preintegration tables and MLS (Moving Least Squares) evaluation for unstructured grids was detailed by [S1, unofficial].

## Sources
[S1] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
