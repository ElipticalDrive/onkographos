# Sphere Tracing

Also called: Hart's Sphere Tracing, SDF Sphere Tracing, SDF Tracing

## Overview
Sphere tracing is an iterative ray-marching algorithm for finding the intersection of a ray with an implicit surface defined by a Signed Distance Function (SDF) [S2][S3, S4, unofficial]. It solves the problem of rendering complex, continuous geometry without explicit polygonal meshes by using the SDF value at the current ray position as a guaranteed lower bound on the distance to the nearest surface [S12, S17, unofficial]. Unlike standard constant-step ray marching, sphere tracing adapts the step size dynamically, allowing the ray to "jump" over empty space efficiently [S3, S30, unofficial]. It is distinguished from its siblings by its strict reliance on the Lipschitz continuity of the distance function to guarantee that no surface is skipped, whereas methods like [Interval Raymarching](13020-interval-raymarching.md) handle uncertainty, and [Step Size Optimization](2054-step-size-optimization.md) focuses on heuristic acceleration [Parent].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, normalized ray direction $\mathbf{d}$, scene SDF function $f(\mathbf{p})$, maximum travel distance $t_{max}$, maximum iteration count $N_{max}$, and convergence threshold $\epsilon$ [S11, S18, S34, unofficial].
*   **Output:** Intersection distance $t$, intersection point $\mathbf{p} = \mathbf{o} + t\mathbf{d}$, and a boolean hit flag [S10, S19, unofficial].

**Data Structures:**
*   **Ray State:** Accumulated distance $t$ (scalar), current position $\mathbf{p}$ (3D vector) [S19, S36, unofficial].
*   **SDF Function:** A mapping $f: \mathbb{R}^3 \to \mathbb{R}$ returning the signed distance to the nearest surface. Positive values indicate outside, negative inside [S2][S23, unofficial].

**Parameters:**
*   $\epsilon$: Convergence threshold. Typical values: $0.00001$ to $0.001$ [S11, S16, S34, unofficial].
*   $N_{max}$: Maximum iterations. Typical values: 10 to 1024 [S12, S29, S34, unofficial].
*   $t_{max}$: Maximum ray distance. Typical values: 10.0 to 1000.0 [S11, S18, S34, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Set accumulated distance $t = 0.0$ (or a small epsilon to avoid division by zero or starting exactly on the surface) [S4, S19, S34, unofficial]. Set iteration counter $i = 0$ [S8, S19, unofficial].
2.  **Position Calculation:** Compute the current point on the ray: $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ [S19, S34, unofficial].
3.  **SDF Evaluation:** Evaluate the scene function at the current point: $d = f(\mathbf{p})$ [S8, S19, S34, unofficial].
4.  **Termination Check (Hit):** If $|d| < \epsilon$ (or $d < \epsilon$ for unsigned distance fields), the ray has converged to the surface. Return $t$ and $\mathbf{p}$ as a hit [S8, S11, S16, S34, unofficial].
5.  **Termination Check (Miss/Resource):** If $t \ge t_{max}$ or $i \ge N_{max}$, the ray has escaped or exceeded resources. Return $t$ and $\mathbf{p}$ as a miss [S11, S18, S19, S34, unofficial].
6.  **Step Advancement:** Advance the ray by the distance estimate: $t \leftarrow t + d$ [S8, S19, S34, unofficial].
    *   *Note:* If $f$ is a general Lipschitz function with constant $\lambda > 1$, the step is scaled: $t \leftarrow t + \frac{|d|}{\lambda}$ [S2, S2054].
7.  **Increment:** Increment iteration counter $i \leftarrow i + 1$ [S19, unofficial].
8.  **Loop:** Repeat from Step 2.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sphere_trace(ray_origin, ray_direction, scene_sdf, max_dist, max_iters, epsilon):
    """
    Performs sphere tracing to find the intersection of a ray with an SDF surface.
    
    Args:
        ray_origin (vec3): The starting point of the ray.
        ray_direction (vec3): The normalized direction vector of the ray.
        scene_sdf (function): A function f(p) returning the signed distance to the nearest surface.
        max_dist (float): Maximum distance the ray can travel.
        max_iters (int): Maximum number of marching steps.
        epsilon (float): Convergence threshold for surface hit.
        
    Returns:
        tuple: (hit: bool, distance: float, point: vec3)
    """
    t = 0.0
    for i in range(max_iters):
        # 1. Calculate current position
        current_pos = ray_origin + ray_direction * t
        
        # 2. Evaluate SDF
        dist = scene_sdf(current_pos)
        
        # 3. Check for surface hit (convergence)
        # Using absolute value for signed distance fields to handle both inside/outside proximity
        if abs(dist) < epsilon:
            return True, t, current_pos
            
        # 4. Check for escape or resource limit
        if t >= max_dist:
            return False, t, current_pos
            
        # 5. Advance ray by the distance estimate
        # For standard SDFs, Lipschitz constant lambda = 1
        t += dist
        
    # 6. Max iterations reached without convergence
    return False, t, current_pos
```

**Edge Cases and Pitfalls:**
*   **Lipschitz Violation:** If the SDF is not Lipschitz continuous with constant $\lambda=1$ (e.g., due to scaling errors or non-distance functions), the step size $d$ may exceed the true distance to the surface, causing the ray to overshoot and miss the intersection [S2][S4, unofficial].
*   **Precision Issues:** Near the surface, floating-point precision errors can cause oscillation or premature termination. Using a relative epsilon (e.g., $0.001 \times t$) can help maintain precision balance at large distances [S19, unofficial].
*   **Inside Geometry:** If the ray starts inside an object ($f(\mathbf{o}) < 0$), standard sphere tracing with $t += d$ will move the ray *away* from the surface (since $d$ is negative). This is correct for finding the exit point but requires careful interpretation of the hit condition [S2][S19, unofficial].

## Subtopics

### [Zeno Implementation](1544-zeno-implementation.md)
A specific software architecture for sphere tracing that minimizes the interface required to define new implicit surfaces. It solves the problem of rapid prototyping for mathematical visualization by allowing new primitives to be added using only a distance bound function, while optionally bypassing explicit normal calculations via numerical gradients. It is distinguished from general sphere tracing implementations by its specific emphasis on the sufficiency of the positive distance bound for visualization and its built-in six-sample gradient approximation.

### [Cone Intersection](1553-cone-intersection.md)
Cone intersection is an antialiasing technique for sphere tracing that represents a pixel as a viewing cone rather than a point ray, allowing symbolic prefiltering to eliminate aliasing artifacts at grazing angles. It differs from standard sphere tracing by detecting intersections with an "inner cover" surface and computing the fractional coverage of the pixel by the implicit surface, whereas siblings like [Segment Tracing](1675-segment-tracing.md) or [Quadric Tracing](1866-quadric-tracing.md) focus on varying step sizes or bounding volumes without necessarily computing area-based coverage fractions.

### [Segment Tracing](1675-segment-tracing.md)
Segment Tracing is a ray marching algorithm that computes local Lipschitz bounds on candidate segments to determine adaptive step sizes, reducing the number of field function queries compared to global Lipschitz bound methods like Sphere Tracing. It works for a particular class of hierarchical skeletal implicit surfaces, leveraging the composition of distance functions and kernel functions to compute directional Lipschitz bounds. Unlike Sphere Tracing, which uses a global bound that limits step size efficiency, Segment Tracing allows larger, safer steps by adapting to the local gradient and field values.

### [Quadric Tracing](1866-quadric-tracing.md)
Quadric tracing is a method to accelerate sphere tracing by precomputing an augmented distance field using quadric surfaces (second-degree algebraic surfaces) to bound or unbound scene geometry, reducing the number of Signed Distance Function (SDF) evaluations. It solves the slowdown of sphere tracing near surfaces and with increasing scene complexity, where function evaluation time and convergence slow drastically. Unlike relaxed sphere tracing (which uses a scalar to determine step size and reverts if too large) or enhanced sphere tracing (which uses previous two radii to approximate planar surfaces), quadric tracing uses precomputed geometric bounds to accelerate convergence. It stops when the computed distance value dips below a threshold, switching to enhanced sphere tracing for final convergence.

### [Pyramid Recursive Ray Marching](3802-pyramid-recursive-ray-marching.md)
Pyramid Recursive Ray Marching is a multi-scale acceleration technique for Signed Distance Field (SDF) ray marching that initializes depth estimates at low resolution and recursively refines them at higher resolutions. It distinguishes itself from standard aggressive ray marching by decoupling the coarse initialization phase from the fine-grained recursive refinement, allowing for larger initial steps in empty space while maintaining precision near surfaces.

### [Over-relaxation Sphere Tracing](3805-over-relaxation-sphere-tracing.md)
Over-relaxation Sphere Tracing accelerates the standard sphere tracing algorithm by stepping further along the ray than the conservative signed distance bound (SDF) allows, controlled by a relaxation parameter $\omega \in [1, 2)$. It reduces the number of SDF evaluations by 20–40% in typical 3D renders. Unlike standard sphere tracing, which guarantees no surface is skipped by stepping exactly the distance estimate, this method risks overshooting the surface; it detects such overshoots via an overlap test between consecutive unbounding spheres and falls back to standard sphere tracing ($\omega=1$) if an intersection might have been missed. It also incorporates screen-space error metrics to select the best intersection candidate when the maximum iteration count is exceeded, preventing artifacts from rays grazing object edges.

### [Enhanced Sphere Tracing](3811-enhanced-sphere-tracing.md)
Enhanced Sphere Tracing is a family of ray-marching optimizations that accelerate convergence toward implicit surfaces defined by Signed Distance Functions (SDFs) by taking steps larger than the strict Lipschitz bound or by predicting optimal step sizes based on local surface geometry. It solves the performance bottleneck of standard sphere tracing, which becomes prohibitively slow when rays graze surfaces at shallow angles or when the surface is locally planar. Unlike standard sphere tracing, which uses the queried distance value as the step size, enhanced variants either over-relax the step size by a constant factor or infer the next step size from the previous two distance queries to approximate the surface as locally flat.

### [Uniform Sphere Tracing](4406-uniform-sphere-tracing.md)
Uniform Sphere Tracing is a volume rendering technique that combines ray marching along a Signed Distance Field (SDF) with optical depth integration to render participating media. It solves the problem of rendering volumetric effects like fog, smoke, or clouds by stepping through a volume, accumulating absorption and scattering, and blending the result with the background. It is distinguished from its parent [Sphere Tracing](2326-sphere-tracing.md) by its specific application to volume rendering with uniform density assumptions and its reliance on fixed or clamped step sizes rather than purely adaptive SDF-driven steps for the optical integration phase, though it uses SDF for surface detection. It differs from [Zeno Implementation](1544-zeno-implementation.md) and [Differentiable Sphere Tracing](2316-differentiable-sphere-tracing.md) by not focusing on differentiability or specific engine integration, but rather on the fundamental algorithmic structure of uniform density volume rendering via ray marching.

### [Synchronized-tracing](8552-synchronized-tracing.md)
Synchronized-tracing is a tile-based rendering pipeline for implicit volumes (blobtrees) that synchronizes threads within a subfrustum to allow coherent field evaluation, using sparse bottom-up tree traversal to prune the blobtree on-the-fly. It is designed for interactive modeling scenarios where no preprocessing is required when primitive parameters are updated. It sets itself apart from standard sphere tracing by introducing a synchronization mechanism via a low-resolution A-buffer to manage thread coherence within workgroups, and by using compact, smooth CSG operators to compute tight bounding volumes for approximate signed distance fields.

### [Cone-Traced Supersampling](9466-cone-traced-supersampling.md)
Cone-Traced Supersampling (CTSS) is an efficient spatial antialiasing solution for Signed Distance Field (SDF) rendering that performs supersampling along the traced ray near surfaces with partial visibility. It identifies object contours by evaluating cone intersections within a pixel's view frustum, avoiding the need for casting additional rays per pixel or offline prefiltering. Subpixel Edge Reconstruction (SER) extends CTSS to locate and resolve complex pixels with geometric edges in relatively flat regions that are otherwise undetected by cone intersections. Unlike conventional supersampling, CTSS minimizes shading computations, and unlike post-processing filters, it avoids undesirable blurriness and ghosting.

### [Kernel-Reflection Sequences](10186-kernel-reflection-sequences.md)
Kernel-Reflection Sequences (KRS) are a procedural modeling technique for generating complex, fractal-like geometries using Signed Distance Functions (SDFs) and reflection transformations. It solves the problem of evaluating global distance functions for recursive structures without recursion, enabling real-time sphere tracing on parallel hardware like GPUs. Unlike standard Iterated Function Systems (IFS) which require recursive traversal and cause divergent control flow, KRS uses a fixed operation count logarithmic time algorithm, allowing efficient parallelization.

### [Analytical and Differentiable Methods](13369-analytical-and-differentiable-methods.md)

### Cube-tracing
An algorithm using the chessboard metric (or Manhattan metric) for distance computation, potentially resulting in more efficiently computed distances and ray intersection. Left as further research [S41].

### Ray Grouping
An optimization that improves performance by lowering the total number of marching steps by decreasing the total number of initial rays, grouping rays together into blocks [S36, unofficial]. Groups rays together during rendering to reduce total execution time [S36, unofficial]. Split groups of rays into smaller groups, though the implementation was noted to be slightly too eager to split groups [S36, unofficial]. Improvements were more significant for higher resolutions, as the cost at each step for a given group is not dependent on the number of rays in the group [S36, unofficial]. Tested on 64x64 and 128x128 pixel displays with 3 spheres at varying distances. Group size 2 lowered execution time. Sizes 4 and 8 did not decrease significantly further. Relative performance improved slightly more on the 128x128 display. Table 5.2 shows clock cycles: 64x64 display drops from 128k (size 1) to 84k (size 2), then 80k/82k. 128x128 drops from 512k to 328k, then 312k/313k [S36, unofficial]. Ray grouping reduces rays with similar execution flows, making it less effective on traditional lockstepped architectures compared to the custom GPU, though still potentially useful [S36, unofficial]. Implemented in the GLSL shader. Group size 1 is equivalent to no grouping [S36, unofficial].

### Bounding Sphere Optimization
An optimization that decreases the number of calculations made in each march by enclosing several objects into one bounding sphere [S36, unofficial]. Groups multiple objects within a single bounding sphere to optimize ray-object intersection tests during sphere tracing [S36, unofficial]. Objects should be bound in an adaptive way, enclosing objects depending on how they are grouped per frame, approximating the sphere definition for increased performance rather than computing it exactly [S36, unofficial]. Performance gain increases as the number of objects inside the sphere increases [S36, unofficial]. Performance gain is situational, depending on enclosed object count, spacing, and fit. In the tested scene, a gain occurs when >3 objects are enclosed. At 10 enclosed objects, FPS increases by 25%. Performance decreases if <4 objects are enclosed [S36, unofficial]. Bounding Spheres can work together with Orthogonal Culling by culling bounding spheres instead of single objects, decreasing the number of orthogonal projections made [S36, unofficial]. Implemented and tested in the GLSL shader. Requires measuring FPS against an identical unoptimized scene [S36, unofficial].

### Hybrid Ray Marching
Integrates ray marching with distance-based empty space leaping for complex scenes, using coarse SDFs to compute large skips in low-density regions before switching to fine-grained tracing near features [S13]. Balances speed and accuracy in multi-material volumes [S13].

### Acceleration Structures for SDF Evaluation
A set of optimization nodes embedded directly in hierarchical construction trees to reduce the computational cost of field function evaluation while preserving Lipschitz or conservative properties [S42, abstract only, unofficial]. Avoid external accompanying data structures such as octrees by embedding acceleration nodes directly in the construction tree [S42, abstract only, unofficial]. Use proxy and continuous level of detail nodes to reduce the overall evaluation cost [S42, abstract only, unofficial]. Apply a normal warping technique to enhance surface details with negligible computational overhead [S42, abstract only, unofficial]. Speeds up signed distance field evaluation by up to three orders of magnitude compared to standard evaluation [S42, abstract only, unofficial]. Applies to both ray-surface intersection computation in sphere tracing applications and polygonization algorithms [S42, abstract only, unofficial].

### Harnack Tracing
A method for visualizing surfaces encoded by harmonic functions (including angle-valued functions and those with singularities) by using conservative Harnack bounds on the growth of harmonic functions to determine step sizes, avoiding numerical issues common to generic root finding methods [S32, unofficial]. Use conservative Harnack bounds on the growth of harmonic functions to determine the step size for each iteration, ensuring the step does not cross a level set boundary [S32, unofficial]. Perform pointwise evaluation of the harmonic function at the current position to check for convergence to the target level set [S32, unofficial]. Unlike sphere tracing which uses distance bounds, Harnack tracing uses bounds on the growth of harmonic functions; it avoids numerical issues common to generic root finding methods and can visualize surfaces directly from point clouds or polygon soup without linear solves or mesh extraction [S32, unofficial].

### Claybook SDF Construction
A method for fully real-time deformable SDF-based worlds, using a global dense SDF with mip levels for acceleration [S43, unofficial]. 1. Construct global SDF. 2. Build 5 mip levels, where each level encodes distance values twice as great as the previous. 3. Use sphere-tracing, sampling coarser mip levels to travel greater distances conservatively. 4. Ascend/descend mip levels based on sampled distance magnitude [S43, unofficial]. Modification: Apply pre-defined 'brushes' to the world SDF. Only nearby affected areas are re-evaluated due to brush locality [S43, unofficial]. Global SDF volume of resolution 1024x1024x512. Five mip levels are constructed. The volume is dense (empty and non-empty regions stored alike) [S43, unofficial]. Compact data structures (like SVOs) are incompatible with this modification method because surface-intersecting voxel counts are not static, requiring slow insertions and indirection tables. Dense volumes allow direct spatial indexing [S43, unofficial]. Used in Claybook (Aaltonen, 2018) [S43, unofficial].

### Conservative Sphere Tracing Step Sizing
A fix for volumetric cloud rendering where sphere tracing takes overly aggressive step sizes, skipping cloud boundaries and leaving circular/scalloped density artifacts [S44, unofficial]. Clamp the pre-computed 2D organic cell SDF in `cloud_weather_bake.comp` to a conservative maximum of `30000.0 * uWorldScale` [S44, unofficial]. Reduce the sphere tracing step size multiplier from `0.8` to `0.4` in `atmosphere_lowres.comp` to make the empty space skipping strictly conservative [S44, unofficial]. Clamp the empty space skipping step size to a maximum of `2000.0 * worldScale` to prevent overly aggressive jumps that bypass cloud details or boundaries [S44, unofficial]. SDF Max Clamp: `30000.0 * uWorldScale` [S44, unofficial]. Step Size Multiplier: `0.4` (reduced from `0.8`) [S44, unofficial]. Step Size Max Clamp: `2000.0 * worldScale` [S44, unofficial]. In empty regions of the weather map, the organic cell SDF returned uncapped values (up to 1e14), causing bilinear filtering to interpolate extremely large values near cloud edges [S44, unofficial]. The empty-space skipping sphere tracing loop used an aggressive step multiplier of 0.8 with no upper bound, resulting in rays jumping past the actual cloud surface [S44, unofficial].

## Operating characteristics
*   **Time Complexity:** The number of iterations depends on the scene geometry and the Lipschitz constant. In empty space, steps are large, leading to $O(1)$ or logarithmic convergence relative to distance. Near surfaces, steps shrink, leading to linear convergence with respect to precision $\epsilon$ [S2][S30, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, as only the current position and distance are stored. The SDF function itself may require $O(N)$ memory if it is a precomputed volume, or $O(1)$ if it is procedural [S1, S23, unofficial].
*   **Performance:** Sphere tracing is highly efficient for simple, smooth implicit surfaces and can run in real-time on GPUs [S23, S27, unofficial]. Performance degrades with complex CSG operations or high-frequency details in the SDF, as the Lipschitz constant may increase or the function evaluation becomes expensive [S1, unofficial][S42, abstract only, unofficial].
*   **Strengths:** No preprocessing required for procedural SDFs; perfect antialiasing potential via cone tracing; compact representation of geometry [S14, S23, unofficial].
*   **Limitations:** Cannot handle non-Lipschitz functions safely; slow convergence near grazing angles or flat surfaces (mitigated by [Enhanced Sphere Tracing](3811-enhanced-sphere-tracing.md)); limited to implicit representations [S1, unofficial][S5].
*   **When to Use:** Real-time rendering of procedural shapes, SDF-based terrain, or implicit volumes where mesh generation is prohibitive [S23, S27, unofficial].
*   **When Not to Use:** Scenes with explicit polygonal geometry (use rasterization or standard ray tracing); scenes with non-smooth or discontinuous distance functions [S23, S33, unofficial].

## Background
Sphere tracing was introduced by John Hart in 1996 in the paper "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" [S14, unofficial][S39]. It was developed to provide a robust and efficient way to render implicit surfaces defined by distance functions, leveraging the geometric property that a sphere of radius $d$ centered at a point $\mathbf{p}$ with distance $d$ to the surface cannot intersect the surface. This method has become a cornerstone of procedural rendering and SDF-based graphics, widely used in real-time shaders and offline rendering for its simplicity and visual quality [S16, S23, unofficial].

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
