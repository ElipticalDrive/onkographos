# Ray Marching

Also called: jittered ray marching

## Overview
Ray marching is an iterative technique for finding intersections with solid geometry or evaluating volumetric fields by stepping along a ray and testing for intersections [S2, S10, unofficial]. It is used when the surface function is not easy to solve analytically or when rendering non-uniform volumetrics, implicit functions, fractals, and parametric surfaces where intersection is not known ahead of time [S2, S10, unofficial]. Unlike classical ray tracing which seeks exact intersections directly, ray marching works iteratively, moving the ray through space in steps until a surface is found, a maximum distance is exceeded, or enough samples are collected for volume integration [S8, S10, unofficial]. It sets itself apart from its parent [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by encompassing a broader family of methods, including adaptive stepping via Signed Distance Functions (SDFs) and fixed-step approaches for volume rendering, whereas the parent specifically refers to the fixed-stride mitigation of root-finding drawbacks [S2, S7, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Ray origin $r_o$ and direction $r_d$ (normalized) [S8, S9, unofficial].
*   Scene representation: A function $scene(p)$ that returns a value at point $p$. This value is either a Signed Distance Function (SDF) for surface rendering or a density/opacity field for volume rendering [S7, S9, unofficial].
*   Parameters:
    *   $u_{maxDis}$: Maximum ray distance (far plane) [S9, unofficial].
    *   $u_{eps}$: Hit threshold (surface proximity) [S9, unofficial].
    *   $u_{maxSteps}$: Maximum number of iterations [S9, unofficial].
    *   $min\_step\_len$, $max\_step\_len$: Bounds for step size in adaptive or constrained fixed-step modes [S1, book].

**Outputs:**
*   For surface rendering: Hit position, surface normal, and final color [S8, S9, unofficial].
*   For volume rendering: Accumulated color and opacity [S4, unofficial].

### Data Structures
No complex data structures are required for the core algorithm itself, as it relies on procedural evaluation. However, for acceleration (space leaping), preprocessed safety distances or grids may be used [S2, S10, unofficial]. For volume rendering, the scene is often represented as a neural network or a 3D texture grid [S3, S6].

### Algorithm Steps
The general iterative process is as follows [S8, S9, unofficial]:

1.  **Ray Generation**: Compute ray origin $r_o$ and direction $r_d$ for the current pixel [S8, unofficial].
2.  **Initialization**: Set total distance traveled $d = 0$ and current position $p = r_o$ [S9, unofficial].
3.  **Iteration Loop**: Repeat for $i = 0$ to $u_{maxSteps}$:
    a.  **Query Scene**: Evaluate the scene function at the current position: $cd = scene(p)$ [S9, unofficial].
        *   If using SDF: $cd$ is the distance to the nearest surface.
        *   If using Volume: $cd$ is the density or transmittance contribution at $p$.
    b.  **Termination Check**:
        *   If $cd < u_{eps}$ (for SDF) or sufficient samples accumulated (for Volume), break loop [S9, unofficial].
        *   If $d \ge u_{maxDis}$, break loop (missed geometry/background) [S9, unofficial].
    c.  **Advance Ray**:
        *   **Sphere Tracing (Adaptive)**: Advance by the SDF value: $d \leftarrow d + cd$ [S7, S9, unofficial]. This is safe because no geometry exists within distance $cd$ [S7, unofficial].
        *   **Fixed Step**: Advance by a constant $\Delta t$: $d \leftarrow d + \Delta t$ [S7, unofficial].
    d.  **Update Position**: $p \leftarrow r_o + d \cdot r_d$ [S9, unofficial].
4.  **Post-Processing**:
    *   If hit ($d < u_{maxDis}$): Compute lighting (diffuse, specular, ambient) using normals derived from numerical differentiation of the SDF [S8, S9, unofficial].
    *   If miss: Output clear/background color [S9, unofficial].
    *   For Volume: Accumulate color and opacity using the volume rendering equation [S4, unofficial].

### Pseudocode
The following pseudocode implements the standard sphere tracing variant, which is the most common form of ray marching for implicit surfaces [S7, S9, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march(ro, rd, scene_func, max_steps, max_dist, epsilon):
    """
    Iteratively march a ray to find intersection with implicit surface defined by scene_func.
    
    Args:
        ro: vec3, ray origin
        rd: vec3, normalized ray direction
        scene_func: function, returns SDF value (distance to nearest surface) for a point
        max_steps: int, maximum number of iterations
        max_dist: float, maximum distance to search
        epsilon: float, threshold for hit detection
        
    Returns:
        float: total distance traveled if hit, max_dist if missed
    """
    d = 0.0  # total distance travelled
    
    for i in range(max_steps):
        # Calculate current position
        p = ro + d * rd
        
        # Query scene distance
        cd = scene_func(p)
        
        # Check for hit or max distance
        if cd < epsilon:
            return d
        if d >= max_dist:
            return max_dist
            
        # Advance ray by the distance to the nearest surface (Sphere Tracing)
        # This is safe because no geometry exists inside the sphere of radius cd
        d += cd
        
    return max_dist

def compute_normal(scene_func, p, h=0.001):
    """
    Compute surface normal via numerical differentiation of SDF.
    """
    # Gradient approximation using central differences
    dx = scene_func(p + vec3(h, 0, 0)) - scene_func(p - vec3(h, 0, 0))
    dy = scene_func(p + vec3(0, h, 0)) - scene_func(p - vec3(0, h, 0))
    dz = scene_func(p + vec3(0, 0, h)) - scene_func(p - vec3(0, 0, h))
    
    normal = normalize(vec3(dx, dy, dz))
    return normal
```

### Formulas
*   **Sphere Tracing Update**: $p_{n+1} = p_n + d(p_n)\hat{d}$, where $p_n$ is current position, $d(p_n)$ is SDF value, $\hat{d}$ is ray direction [S7, unofficial].
*   **Sphere SDF**: $distance = \|p\| - r$ [S8, unofficial].
*   **Cube SDF**: $distance = \max(|p.x| - s, |p.y| - s, |p.z| - s)$, where $s$ is half-size [S8, unofficial].
*   **Union SDF**: $distance = \min(distance_1, distance_2)$ [S8, unofficial].
*   **Jensen's Inequality Bias (Volume)**: Assuming constant extinction leads to bias: $\exp(\mathbb{E}[-\tau]) \le \mathbb{E}[\exp(-\tau)]$ [S11].

### Parameters
*   $u_{eps}$: Hit threshold. Typical value: $0.001$ [S8, unofficial].
*   $min\_step\_len$, $max\_step\_len$: Used in constrained marching to prevent overly small or large steps [S1, book].

### Edge Cases and Pitfalls
*   **Overestimation**: Sphere tracing requires the SDF to be conservative (never overestimate true distance). If the distance function is optimistic, the ray can step past geometry and miss it [S7, unofficial].
*   **Precision**: Insufficient precision can lead to "holes" or incorrect intersections [S8, unofficial].
*   **Volume Bias**: Fixed-step ray marching for volumes ignores true heterogeneity, leading to bias characterized by Jensen's inequality. This can be reduced by smaller strides (conforming to Nyquist frequency) but reduces performance [S11].
*   **Performance at Low Samples**: Ray marching performs poorly at lower sample counts for volume rendering compared to distance sampling [S6].

## Subtopics

### [Sphere Tracing](52-name.md)
Uses SDFs for adaptive stepping, jumping through empty space. Choose when geometry is defined by implicit functions.

### [Fixed-Step Ray Marching](1080-standard-constant-step-ray-marching.md)
Uses constant $\Delta t$. Choose for simple volumetric integration or when SDF is unavailable.

### [Jittered Ray Marching](1244-iteration-based-coloring.md)
Adds random offsets to fixed steps to reduce aliasing in volume rendering. Choose for anti-aliasing in volumetrics.

## Operating characteristics
*   **Complexity**: High computational complexity due to many iterations required per pixel [S8, unofficial].
*   **Performance**: On an RTX 8000 GPU at 1024 spp, ray marching on a room scene took $11.0 \pm 0.5$ ms. On a GTX 980 Ti at 256 spp, it took $10.3 \pm 0.2$ ms [S6].
*   **Scaling**: Scalability to large explicit scenes is poor unless acceleration structures (BVH, grids) are used for distance queries [S7, unofficial].
*   **Strengths**: Natural for procedural geometry, compact scene representation, easy integration of global illumination and soft shadows [S8, unofficial].
*   **Limitations**: Inefficient for explicit triangle meshes without SDF conversion; poor convergence at low sample counts for volumes [S6][S7, unofficial].
*   **When to Use**: Rendering implicit surfaces, fractals, volumetrics, and parametric surfaces where analytical intersection is difficult [S2, S10, unofficial].

## Background
Ray marching was first encountered by the author in the context of per-pixel displacement mapping techniques, specifically relief mapping of non-height-field surface details [S2, unofficial]. It is a technique similar to traditional ray tracing but relies on iterative stepping rather than direct intersection solving [S2, unofficial]. It is widely used in real-time differentiable volume rendering (e.g., NeRF) and procedural graphics/shaders (e.g., ShaderToy) [S3][S8, unofficial].

## Sources
[S1] Rendering with mental ray, https://archive.org/details/renderingwithmen0000drie, fetched 2026-10-25 (book)
[S2] What Is Ray Marching?, https://computergraphics.stackexchange.com/questions/161/what-is-ray-marching-is-sphere-tracing-the-same-thing, fetched 2026-10-26 (unofficial)
[S3] Jrender: An efficient differentiable rendering library based on Jittor, https://doi.org/10.1016/j.gmod.2023.101202, fetched 2026-10-27
[S4] NeRF Explained Simply | The 5D Function & Volumetric Rendering, https://www.youtube.com/watch?v=BL-KcFm4Wms, fetched 2026-10-27 (unofficial)
[S5] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
[S6] Collimated Whole Volume Light Scattering in Homogeneous Finite Media, https://doi.org/10.1109/tvcg.2021.3135764, fetched 2026-10-27
[S7] ray marching vs ray tracing, https://tutorials.tektite.studio/ray-marching/ray-marching-vs-ray-tracing/, fetched 2026-10-27 (unofficial)
[S8] What is Ray Marching and how does it work? Ray Marching is one of the most flexible visualization techniques, allowing you to create complex 3D scenes without polygons. Instead of searching for exact , https://serverspace.us/support/help/what-is-ray-marching-and-how-does-it-work/, fetched 2026-10-27 (unofficial)
[S9] Ray Marching with Three.js Guide, https://www.scribd.com/document/882338593/Ray-Marching-in-Three-js, fetched 2026-10-28 (unofficial)
[S10] What is Ray Marching? Is Sphere Tracing the same thing?, https://computergraphics.stackexchange.com/questions/161, fetched 2026-10-29 (unofficial)
[S11] PDF Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
