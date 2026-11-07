# Standard Constant Step Ray Marching

Also called: Constant step ray marching, March Calculation

## Overview
Standard Constant Step Ray Marching is an iterative technique for finding the intersection of a ray with implicit surfaces or volumetric media by advancing the ray along its direction by a fixed distance $\Delta t$ at each iteration [S13, S19, S28, unofficial]. Unlike [Sphere Tracing](1674-sphere-tracing.md), which uses the local distance field value as the step size to guarantee no overshooting, this method uses a uniform step size, requiring careful selection to avoid missing thin geometry or aliasing high-frequency details [S19, S28, unofficial]. It solves the problem of rendering implicit surfaces or volumes without explicit polygonal meshes by discretizing the ray path into uniform intervals [S13, S28, unofficial].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and unit direction $\mathbf{d}$ [S19, unofficial].
*   Fixed step size $\Delta t$ [S19, unofficial].
*   Maximum travel distance $t_{max}$ (far clipping plane) [S19, unofficial].
*   Maximum number of iterations $N_{max}$ [S19, unofficial].
*   Surface/Hit threshold $\epsilon$ (precision) [S19, unofficial].
*   Scene function $f(\mathbf{p})$ which returns a scalar value (e.g., distance to surface, density, or boolean inside/outside) [S19, S28, unofficial].

**Outputs:**
*   Intersection distance $t$ (if hit) or miss status [S19, unofficial].
*   Optional: Surface normal, material ID, or accumulated radiance [S20, S23, unofficial].

**Data Structures:**
*   `ray`: Struct containing `org` (vec3), `dir` (vec3) [S20, unofficial].
*   `sceneHit` (optional): Struct containing `position` (vec3), `normal` (vec3), `distanceTravelled` (float), `hit` (bool) [S4, unofficial].

**Algorithm:**
1.  Initialize accumulated distance $t = 0$ [S19, unofficial].
2.  Initialize step counter $k = 0$ [S19, unofficial].
3.  Loop while $k < N_{max}$ AND $t < t_{max}$:
    a. Calculate current position: $\mathbf{p}_k = \mathbf{o} + t \mathbf{d}$ [S19, unofficial].
    b. Evaluate scene function: $val = f(\mathbf{p}_k)$ [S19, unofficial].
    c. Check termination conditions:
        i. If $val \le 0$ (for boolean/implicit) OR $|val| < \epsilon$ (for distance fields), the ray has hit or entered the surface. Terminate and return $t$ (or refined $t$) [S19, S28, unofficial].
        ii. If $t \ge t_{max}$, the ray has escaped. Terminate and return miss [S19, unofficial].
    d. Advance ray: $t \leftarrow t + \Delta t$ [S19, unofficial].
    e. Increment counter: $k \leftarrow k + 1$ [S19, unofficial].
4.  If loop completes without hit, return miss [S19, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def standard_constant_step_ray_march(ray_origin, ray_direction, step_size, max_dist, max_steps, epsilon, scene_function):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    step_size: float (Delta t)
    max_dist: float (t_max)
    max_steps: int (N_max)
    epsilon: float (precision threshold)
    scene_function: function(p) -> float (returns distance or density)
    """
    t = 0.0
    for k in range(max_steps):
        if t >= max_dist:
            return None  # Miss: exceeded max distance
        
        # Calculate current position
        p = ray_origin + ray_direction * t
        
        # Evaluate scene function
        # For SDF: returns distance to nearest surface
        # For Boolean: returns 1 if outside, 0 if inside (or similar)
        val = scene_function(p)
        
        # Check for hit
        # If using SDF: hit if distance is very small
        # If using Boolean/Density: hit if val <= 0 or density > threshold
        if val <= epsilon:
            return t  # Hit at distance t
        
        # Advance step
        t += step_size
        
    return None  # Miss: exceeded max steps
```

**Parameters:**
*   $\Delta t$: Fixed step size. Typical values range from $0.001$ to $0.1$ depending on scene scale and desired accuracy [S6, S13, S23, unofficial]. Smaller $\Delta t$ increases accuracy but requires more steps [S14].
*   $\epsilon$: Precision threshold. Typical value: $0.001$ [S8, S20, S27, unofficial].
*   $t_{max}$: Maximum ray distance. Typical value: $100.0$ to $400.0$ [S6, S7, unofficial].
*   $N_{max}$: Maximum iterations. Typical value: $64$ to $255$ [S8, S11, S17, unofficial].

**Edge Cases and Pitfalls:**
*   **Overshooting:** If $\Delta t$ is too large relative to the smallest feature size in the scene, the ray may jump over thin geometry or surfaces, resulting in false misses [S19, S28, unofficial].
*   **Aliasing:** Large step sizes can cause jagged edges or missing details in high-frequency implicit surfaces [S19, unofficial].
*   **Performance:** Unlike Sphere Tracing, constant step marching does not adapt to local geometry, potentially taking many small steps in empty space or large steps that skip details [S19, unofficial].
*   **Stagnation:** If the ray is parallel to a surface or stuck in a region where the function value doesn't change significantly, it may waste iterations [S19, unofficial].

## Subtopics

### Coarse-to-Fine Entry Point Search
An optimization for determining the entry point into solid matter more accurately. It samples forward along the ray in large steps until 'solid matter' is hit, then samples backward in smaller steps to refine the entry point [S31, unofficial].

### Quilez Ray Marching AO
A modification of Evans' method using a ray equation for sampling, allowing for constant step marching along the normal to compute ambient occlusion. The occlusion formula is $AO = 1 - \frac{1}{2} \sum_{k=1}^{k_{max}} \frac{1}{k} (k \Delta t - f(r_n(k)))$ [S28, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of steps taken, bounded by $N_{max}$ [S19, unofficial].
*   **Memory Complexity:** $O(1)$ per ray, as it only stores current position and accumulated distance [S19, unofficial].
*   **Performance:** Performance is highly dependent on the choice of $\Delta t$. Too small, and it becomes slow; too large, and it misses geometry [S19, unofficial]. It is generally slower than Sphere Tracing for SDFs because it cannot take advantage of the distance field's Lipschitz property to skip empty space efficiently [S19, unofficial].
*   **Strengths:** Simple to implement; works with any scalar field function (not just SDFs); suitable for volumetric rendering where density varies continuously [S19, S23, unofficial].
*   **Limitations:** Prone to overshooting thin geometry; requires manual tuning of step size; less efficient for SDFs compared to adaptive step methods [S19, S28, unofficial].
*   **When to use:** When rendering volumetric data with known density fields; when the scene function is not a valid SDF (e.g., boolean functions); when simplicity of implementation is prioritized over performance [S13, S19, unofficial].
*   **When not to use:** When rendering complex SDF scenes with thin features; when performance is critical and adaptive stepping is feasible [S19, unofficial].

## Background
Ray marching was introduced as an alternative to classic ray-tracing, taking a series of finite steps along a ray until an object is struck or the permitted number of steps is exceeded [S15, S16, unofficial]. The concept of using distance fields for ray marching was formalized by John Hart in 1995 with "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" [S3, unofficial]. Standard constant step marching is a simpler variant that predates or exists alongside adaptive methods, often used in early implicit surface rendering and educational contexts [S13, S19, unofficial]. It is widely used in demoscene graphics and procedural rendering for its simplicity [S1, S13, unofficial].

## Sources
[S1] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S2] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S3] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
[S4] AustinGoodman/non-linear-sphere-tracing, https://github.com/AustinGoodman/non-linear-sphere-tracing, fetched 2026-10-26 (unofficial)
[S5] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S6] Adam-Gleave/sdf-terrain, https://github.com/Adam-Gleave/sdf-terrain, fetched 2026-10-26 (unofficial)
[S7] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S8] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S9] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S10] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
[S11] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S12] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S13] Spheretracing Basics | Ronja's tutorials, https://www.ronja-tutorials.com/post/043-spheretracing-basics/, fetched 2026-10-27 (unofficial)
[S14] Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere ..., https://diglib.eg.org/items/7f21e58c-639e-4359-bf94-414738612006, fetched 2026-10-27
[S15] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1819/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S16] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S17] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S18] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S19] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S20] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S21] jk9g13/LaFortuna-SDF-Ray-marching, https://github.com/jk9g13/LaFortuna-SDF-Ray-marching, fetched 2026-10-27 (unofficial)
[S22] Dionysos199/Ray-marching, https://github.com/Dionysos199/Ray-marching, fetched 2026-10-27 (unofficial)
[S23] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S24] Sample - Ray Marching and Signed Distance Functions, https://github.com/ComputeNode/cyfra/pull/13, fetched 2026-10-27 (unofficial)
[S25] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
[S26] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S27] GPU Ray Marching Techniques, https://www.scribd.com/document/814069988/5-GPU-Ray-Marching, fetched 2026-10-28 (unofficial)
[S28] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S29] SDF Raymarching, https://zone.dog/braindump/sdf_marching/, fetched 2026-10-29 (unofficial)
[S30] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
[S31] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
