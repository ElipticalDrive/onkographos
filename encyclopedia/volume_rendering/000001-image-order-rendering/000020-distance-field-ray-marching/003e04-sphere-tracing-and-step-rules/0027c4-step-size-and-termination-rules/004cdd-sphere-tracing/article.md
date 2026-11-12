# Sphere Tracing

Also called: Hart's Sphere Tracing, SDF Sphere Tracing, SDF Tracing

## Overview
Sphere Tracing is an iterative ray-marching algorithm for rendering implicit surfaces defined by Signed Distance Functions (SDFs) [S1, unofficial][S2][S3, S4, unofficial]. It solves the problem of finding the intersection between a camera ray and a complex, procedurally defined geometry without explicit polygonal meshes [S16, S17, unofficial]. Unlike standard constant-step ray marching, Sphere Tracing uses the SDF value at the current position as a safe lower bound for the distance to the nearest surface, allowing the ray to "leap" through empty space in large steps [S12, S17, unofficial]. This guarantees that the ray never overshoots or penetrates the surface, provided the function satisfies the Lipschitz condition with constant $\lambda=1$ [S2, S39]. It sets itself apart from siblings like [Interval Raymarching](13020-interval-raymarching.md) by relying on point-wise distance estimates rather than interval arithmetic, and from [Step Size Optimization](2054-step-size-optimization.md) by using the strict geometric bound rather than heuristic or historical approximations for step sizing.

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, normalized ray direction $\mathbf{d}$, maximum traversal distance $t_{max}$, maximum iteration count $N_{max}$, and convergence threshold $\epsilon$.
*   **Output:** Intersection distance $t$, hit mask (boolean), and optionally the surface normal and material ID.

**Data Structures:**
*   **Ray:** Struct or tuple containing $\mathbf{o} \in \mathbb{R}^3$ and $\mathbf{d} \in \mathbb{R}^3$ ($\|\mathbf{d}\|=1$).
*   **SDF Function:** $f: \mathbb{R}^3 \to \mathbb{R}$, returning the signed distance to the nearest surface. Positive outside, negative inside, zero on the surface [S2][S32, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Set accumulated distance $t = 0.0$ (or a small epsilon $t_{min}$ to avoid division by zero or immediate termination if the ray starts on the surface) [S4, S18, S34, unofficial]. Set iteration counter $i = 0$.
2.  **Position Calculation:** Compute the current 3D position on the ray: $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ [S10, S18, unofficial].
3.  **SDF Evaluation:** Query the scene function to get the distance to the nearest surface: $d = f(\mathbf{p})$ [S6, S21, unofficial].
4.  **Termination Check (Hit):** If $|d| < \epsilon$ (or $d < \epsilon$ for unsigned distance fields), the ray has converged to the surface. Terminate and return $t$ [S1, S16, S34, unofficial]. Note: Some implementations check if the ray has moved from outside to inside ($d < 0$) to detect penetration [S1, unofficial].
5.  **Termination Check (Miss):** If $t > t_{max}$, the ray has exited the scene. Terminate and return miss status [S18, S34, unofficial].
6.  **Termination Check (Limit):** If $i \ge N_{max}$, terminate to prevent infinite loops [S1, S8, unofficial].
7.  **Step Update:** Advance the ray by the distance estimate: $t \leftarrow t + d$ [S10, S19, S30, unofficial].
    *   *Safety Note:* If the SDF is not a true SDF but a Lipschitz function with constant $\lambda > 1$, the step must be scaled: $\Delta t = \frac{|d|}{\lambda}$ [S2, S2054].
8.  **Increment:** $i \leftarrow i + 1$. Repeat from Step 2.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_trace(ray_origin, ray_direction, sdf_function, max_dist, max_iters, epsilon):
    """
    Performs sphere tracing for a single ray.
    
    Args:
        ray_origin (vec3): The starting point of the ray.
        ray_direction (vec3): The normalized direction vector of the ray.
        sdf_function (callable): A function f(p) returning signed distance to surface.
        max_dist (float): Maximum distance the ray can travel.
        max_iters (int): Maximum number of marching steps.
        epsilon (float): Convergence threshold for surface hit.
        
    Returns:
        tuple: (hit: bool, distance: float, point: vec3, normal: vec3)
    """
    t = 0.0
    point = ray_origin
    
    for i in range(max_iters):
        # 1. Evaluate SDF at current position
        dist = sdf_function(point)
        
        # 2. Check for surface hit (convergence)
        # Using absolute value for signed distance, or just dist < epsilon for unsigned
        if abs(dist) < epsilon:
            # Compute normal via central differences if needed
            normal = compute_normal(sdf_function, point)
            return True, t, point, normal
        
        # 3. Check for miss (exceeded max distance)
        if t > max_dist:
            return False, t, point, vec3(0, 0, 0)
        
        # 4. Step forward by the distance estimate
        # The SDF value guarantees no surface is closer than 'dist'
        t += dist
        point = ray_origin + ray_direction * t
        
    # 5. Max iterations reached
    return False, t, point, vec3(0, 0, 0)

def compute_normal(sdf_function, point, step=0.001):
    """
    Computes the surface normal using central finite differences.
    """
    dx = vec3(step, 0, 0)
    dy = vec3(0, step, 0)
    dz = vec3(0, 0, step)
    
    nx = sdf_function(point + dx) - sdf_function(point - dx)
    ny = sdf_function(point + dy) - sdf_function(point - dy)
    nz = sdf_function(point + dz) - sdf_function(point - dz)
    
    return normalize(vec3(nx, ny, nz))
```

**Parameters and Typical Values:**
*   $\epsilon$ (Convergence Threshold): Typically $0.001$ to $0.0001$ [S11, S16, S34, unofficial]. Some implementations use a relative threshold, e.g., $0.001 \times t$ [S19, unofficial].
*   $N_{max}$ (Max Iterations): Typically $10$ to $255$ for real-time, up to $1024$ for high-quality [S1, S12, S18, S29, S34, unofficial].
*   $t_{max}$ (Max Distance): Scene-dependent, often $100.0$ or $1000.0$ [S11, S18, S34, unofficial].

**Edge Cases and Pitfalls:**
*   **Lipschitz Violation:** If the SDF does not satisfy the Lipschitz condition with $\lambda=1$ (i.e., the gradient magnitude exceeds 1), the step size $d$ is no longer a safe lower bound, and the ray may overshoot the surface [S2, S2054].
*   **Precision Errors:** Near the surface, floating-point precision can cause oscillation or premature termination. Using a relative epsilon or clamping the minimum step size can mitigate this [S19, S44, unofficial].
*   **Inside the Surface:** If the ray starts inside an object ($f(\mathbf{o}) < 0$), standard sphere tracing may step away from the surface or fail to converge if not handled. Some implementations check for sign changes (outside to inside) to detect hits [S1, unofficial].
*   **Gradients:** Normals are typically computed via numerical gradients (finite differences), which adds 6 extra SDF evaluations per hit point [S11, S18, unofficial]. Analytical derivatives are preferred if available [S45, unofficial].

## Subtopics

### [Zeno Implementation](1544-zeno-implementation.md)
A software architecture minimizing the interface for new implicit surfaces, emphasizing the sufficiency of positive distance bounds and built-in gradient approximation.

### [Cone Intersection](1553-cone-intersection.md)
An antialiasing technique representing pixels as viewing cones for symbolic prefiltering, differing from standard point-ray sphere tracing.

### [Segment Tracing](1675-segment-tracing.md)
Computes local Lipschitz bounds on segments for adaptive step sizes, reducing field queries compared to global bounds.

### [Quadric Tracing](1866-quadric-tracing.md)
Accelerates tracing by precomputing augmented distance fields using quadric surfaces to bound geometry.

### [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md)
Enables gradient back-propagation through the tracing process for neural implicit surfaces.

### [Chamfer-L1 Distance Computation](2752-chamfer-l1-distance-computation.md)
A post-processing metric for evaluating SDF fidelity against ground-truth meshes.

### [Raymarched SDF Volume Rendering](3663-raymarched-sdf-volume-rendering.md)
Renders 3D scenes defined by implicit surfaces without rasterization, focusing on the marching loop and shading.

### [Pyramid Recursive Ray Marching](3802-pyramid-recursive-ray-marching.md)
Multi-scale acceleration initializing depth estimates at low resolution and refining recursively.

### [Over-relaxation Sphere Tracing](3805-over-relaxation-sphere-tracing.md)
Steps by a factor $\omega > 1$ times the SDE, falling back to standard tracing if overshoot is detected.

### [Enhanced Sphere Tracing](3811-enhanced-sphere-tracing.md)
Takes steps larger than the Lipschitz bound or predicts optimal steps based on local geometry to accelerate convergence.

### [Sphere Tracing with Composite SDF and Adaptive Sampling](3867-sphere-tracing-with-composite-sdf-and-adaptive-sampling.md)
Renders composite scenes (primitives, text, arcs) with specialized distance queries and adaptive sampling.

### [Non-Linear Sphere Tracing](4079-non-linear-sphere-tracing.md)
Renders deformed SDFs by formulating traversal as an ODE initial value problem.

### [Uniform Sphere Tracing](4406-uniform-sphere-tracing.md)
Combines ray marching with optical depth integration for participating media with uniform density.

### [Menger Sponge Raymarching](4875-menger-sponge-raymarching.md)
Renders a Menger sponge fractal using a recursive SDF and fixed-step iteration.

### [Synchronized-tracing](8552-synchronized-tracing.md)
Tile-based pipeline for implicit volumes synchronizing threads for coherent field evaluation.

### [Cone-Traced Supersampling](9466-cone-traced-supersampling.md)
Spatial antialiasing performing supersampling along the traced ray near surfaces with partial visibility.

### [Kernel-Reflection Sequences](10186-kernel-reflection-sequences.md)
Procedural modeling technique for fractal geometries using reflection transformations, enabling parallel sphere tracing.

### Cube-tracing
An algorithm using the chessboard (Manhattan) metric for distance computation, potentially resulting in more efficiently computed distances and ray intersection [S41]. Left as further research.

### Ray Grouping
An optimization that improves performance by grouping rays together into blocks to lower the total number of marching steps [S36, unofficial]. Tested on 64x64 and 128x128 displays, group size 2 lowered execution time significantly (e.g., 128k to 84k clock cycles on 64x64), with diminishing returns for sizes 4 and 8 [S36, unofficial].

### Bounding Sphere Optimization
An optimization that decreases calculations by enclosing several objects into one bounding sphere [S36, unofficial]. Performance gain increases with the number of enclosed objects (e.g., 25% FPS increase at 10 objects), but decreases if fewer than 4 objects are enclosed [S36, unofficial].

### Hybrid Ray Marching
Integrates ray marching with distance-based empty space leaping for complex scenes, using coarse SDFs to compute large skips in low-density regions before switching to fine-grained tracing near features [S13].

### Acceleration Structures for SDF Evaluation
Optimization nodes embedded directly in hierarchical construction trees to reduce computational cost of field function evaluation while preserving Lipschitz properties [S42, abstract only, unofficial]. Speeds up SDF evaluation by up to three orders of magnitude [S42, abstract only, unofficial].

### Harnack Tracing
A method for visualizing surfaces encoded by harmonic functions using conservative Harnack bounds on growth to determine step sizes, avoiding numerical issues of generic root finding [S32, unofficial].

### Claybook SDF Construction
A method for real-time deformable SDF-based worlds using a global dense SDF with mip levels for acceleration [S43, unofficial]. Samples coarser mip levels for larger conservative steps [S43, unofficial].

### Aeva Analytical SDF Rendering
Avoids discretization by sphere-tracing analytical SDFs at render time, using offline compilation of CSG trees into optimized shaders [S43, unofficial].

### Conservative Sphere Tracing Step Sizing
A fix for volumetric cloud rendering where sphere tracing takes overly aggressive step sizes, skipping cloud boundaries [S44, unofficial]. Involves clamping the SDF max value and reducing the step size multiplier (e.g., from 0.8 to 0.4) [S44, unofficial].

### Analytical Derivatives for Normals and Slopes
A technique using analytical derivatives of noise to compute fast normals and slopes, adjusting step size based on slope [S45, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K)$, where $N$ is the number of pixels and $K$ is the average number of iterations per ray. $K$ depends on the scene complexity and the smoothness of the SDF. In empty space, $K$ is small; near complex surfaces, $K$ increases [S1, S12, unofficial].
*   **Memory Complexity:** $O(1)$ per ray, as it only stores the current position and distance. The SDF function itself may require significant memory if it involves large textures or neural networks [S1, S10, unofficial].
*   **Measured Performance:** Performance varies widely. On a 64x64 display, standard sphere tracing took ~128k clock cycles, while ray grouping optimization reduced this to 84k [S36, unofficial]. For complex scenes, the number of iterations can exceed 1000, impacting real-time performance [S1, S12, unofficial].
*   **Scaling:** Scales well with parallelization (GPU) as each ray is independent [S1, S10, unofficial]. However, divergence in iteration counts across threads can reduce efficiency [S36, unofficial].
*   **Strengths:** No preprocessing required for simple primitives; infinite sharpness for analytic SDFs; robust handling of complex CSG operations [S3, S16, S23, unofficial].
*   **Limitations:** Slow convergence near grazing angles or flat surfaces; sensitive to Lipschitz constant violations; requires numerical gradients for normals, adding cost [S1, unofficial][S5][S11, unofficial].
*   **When to Use:** Ideal for procedural geometry, fractals, and real-time shader-based rendering where explicit mesh storage is undesirable [S16, S23, unofficial].
*   **When Not to Use:** Not suitable for scenes with highly non-Lipschitz functions or when exact analytical normals are required without numerical approximation [S2][S45, unofficial].

## Background
Sphere Tracing was introduced by John Hart in 1996 in the paper "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" [S14, unofficial][S39]. It was developed to provide an efficient and antialiased method for rendering implicit surfaces defined by distance functions. The method leverages the geometric property that a sphere of radius $d$ centered at a point $\mathbf{p}$ with SDF value $d$ cannot intersect the surface, allowing for safe step sizes [S14, unofficial][S39]. It has since become a foundational technique in real-time graphics, particularly in shader-based rendering and procedural modeling [S16, S23, unofficial]. Variants and extensions have been developed to address its limitations, such as enhanced sphere tracing, differentiable sphere tracing, and cone tracing [S1, unofficial][S15][S14, unofficial].

## Sources
[S1] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
[S2] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S3] What Is Ray Marching?, https://computergraphics.stackexchange.com/questions/161/what-is-ray-marching-is-sphere-tracing-the-same-thing, fetched 2026-10-26 (unofficial)
[S4] Two different, seemingly equivalent implementations of sphere tracing acting differently, https://computergraphics.stackexchange.com/questions/14565/two-different-seemingly-equivalent-implementations-of-sphere-tracing-acting-dif, fetched 2026-10-26 (unofficial)
[S5] Rendering piecewise approximations of SDFs through analytic intersections, https://doi.org/10.1016/j.cag.2024.103981, fetched 2026-10-27
[S6] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S7] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S8] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S9] AlerianEmperor/SDF-Sphere-Tracing, https://github.com/AlerianEmperor/SDF-Sphere-Tracing, fetched 2026-10-26 (unofficial)
[S10] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S11] Gomosuza/ray-marching, https://github.com/Gomosuza/ray-marching, fetched 2026-10-26 (unofficial)
[S12] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S13] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S14] [PDF] A Geometric Method for the Antialiased Ray Tracing of Implicit ..., https://graphics.stanford.edu/courses/cs348b-20-spring-content/uploads/hart.pdf, fetched 2026-10-26 (unofficial)
[S15] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
[S16] What is sphere tracing and how It works | 3D rendering guide, https://serverspace.us/support/help/what-is-sphere-tracing-and-how-does-it-work/, fetched 2026-10-27 (unofficial)
[S17] Spheretracing Basics | Ronja's tutorials, https://www.ronja-tutorials.com/post/043-spheretracing-basics/, fetched 2026-10-27 (unofficial)
[S18] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S19] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S20] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1819/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S21] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S22] Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, https://github.com/Fhz2346/Neural-Volume-Rendering-and-Surface-Rendering, fetched 2026-10-27 (unofficial)
[S23] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S24] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S25] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S26] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
[S27] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S28] aparis69/Segment-Tracing, https://github.com/aparis69/Segment-Tracing, fetched 2026-10-27 (unofficial)
[S29] Sphere Tracing and Ray Marching, https://deepwiki.com/ronja-tutorials/ShaderTutorials/5-sphere-tracing-and-ray-marching, fetched 2026-10-27 (unofficial)
[S30] ray marching vs ray tracing, https://tutorials.tektite.studio/ray-marching/ray-marching-vs-ray-tracing/, fetched 2026-10-27 (unofficial)
[S31] https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/043_SphereTracingBasics/SphereTracingBasics.shader, https://github.com/ronja-tutorials/ShaderTutorials/blob/master/Assets/043_SphereTracingBasics/SphereTracingBasics.shader, fetched 2026-10-28 (unofficial)
[S32] (PDF) Accelerating Sphere Tracing - ResearchGate, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S33] Differentiable Ray Marching: Definition & How It Works, https://inferensys.com/glossary/neural-radiance-fields-and-spatial-computing/implicit-surface-representations/differentiable-ray-marching, fetched 2026-10-28 (unofficial)
[S34] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S35] What is Ray Marching? Is Sphere Tracing the same thing?, https://computergraphics.stackexchange.com/questions/161, fetched 2026-10-29 (unofficial)
[S36] Sphere Tracing GPU An evaluation of the Sphere Tracing algorithm and a GPU designed to run it, https://odr.chalmers.se/items/37b0e53f-fc76-4b0e-96d8-8e864573ada9/full, fetched 2026-10-29 (unofficial)
[S37] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S38] Csaba Bálint 1. Introduction 2. Sphere Tracing 3. Relaxed ..., https://people.inf.elte.hu/csabix/publications/articles/WSPS_2019_Poster.pdf, fetched 2026-10-29 (unofficial)
[S39] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
[S40] Modelado de geometría mediante "Signed Distance Functions" y desarrollo de algoritmos para su intersección, https://core.ac.uk/download/478827061.pdf, fetched 2026-10-30 (unofficial)
[S41] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S42] Enhanced Sphere Tracing | Request PDF - ResearchGate, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
[S43] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S44] Fix volumetric cloud SDF sphere march step sizing, https://github.com/ricecake/boidsish/pull/659, fetched 2026-10-26 (unofficial)
[S45] Raymarching Distance Fields, https://iquilezles.org/articles/raymarchingdf/, fetched 2026-10-27 (unofficial)
