# SDF Normal Estimation via Finite Differences

Also called: Finite Difference Normal Estimation, Inigo Quilez's Normal Method, Normal Estimation via Finite Differences, SDF Gradient, SDF Gradient Normal, SDF Gradient Normal Estimation via Finite Differences, View Space Normal Estimation

## Overview
SDF Normal Estimation via Finite Differences computes the surface normal at a point on a Signed Distance Field (SDF) by numerically approximating the gradient of the distance function $f(\mathbf{p})$ and normalizing the result [S2, S4, S5, unofficial]. This method solves the problem of shading implicit surfaces that lack explicit mesh topology or analytic derivatives, which is common when SDFs are constructed from Boolean combinations of primitives or neural networks [S33, S40, unofficial]. It is distinguished from sibling methods like [Tetrahedral Normal Estimation](1986-tetrahedral-normal-estimation.md) by using axis-aligned offsets rather than geometric stencils, and from [Normal Regularization](3704-normal-regularization.md) by relying on raw numerical gradients without post-processing smoothing [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{p} = (x, y, z)$ on or near the SDF surface (typically where $f(\mathbf{p}) \approx 0$) and a step size $\epsilon$ [S4, S32, unofficial].
*   **Output:** A unit normal vector $\mathbf{n}$ [S2, S32, unofficial].

**Data Structures:**
No specific data structures are required beyond the SDF evaluation function `SDF(p)` and standard 3D vector types. The step size $\epsilon$ is a scalar parameter [S32, unofficial].

**Steps:**
1.  **Define Step Size:** Select a small positive scalar $\epsilon$ [S4, S32, unofficial]. Typical values range from $10^{-4}$ to $10^{-3}$ depending on the scale of the scene [S12, S27, S34, unofficial].
2.  **Evaluate Axis Offsets:** Query the SDF at six points offset from $\mathbf{p}$ along the coordinate axes:
    *   $d_{x+} = f(\mathbf{p} + (\epsilon, 0, 0))$
    *   $d_{x-} = f(\mathbf{p} - (\epsilon, 0, 0))$
    *   $d_{y+} = f(\mathbf{p} + (0, \epsilon, 0))$
    *   $d_{y-} = f(\mathbf{p} - (0, \epsilon, 0))$
    *   $d_{z+} = f(\mathbf{p} + (0, 0, \epsilon))$
    *   $d_{z-} = f(\mathbf{p} - (0, 0, \epsilon))$
    [S1][S12, S33, unofficial].
3.  **Compute Gradient Components:** Calculate the partial derivatives using central differences. The division by $2\epsilon$ is often omitted in implementation because it cancels out during normalization, but the mathematical definition includes it [S32, S46, unofficial]:
    $$
    \begin{aligned}
    g_x &= d_{x+} - d_{x-} \\
    g_y &= d_{y+} - d_{y-} \\
    g_z &= d_{z+} - d_{z-}
    \end{aligned}
    $$
    [S12, S30, S34, unofficial].
4.  **Normalize:** Construct the gradient vector $\mathbf{g} = (g_x, g_y, g_z)$ and normalize it to obtain the normal $\mathbf{n}$ [S2, S32, unofficial]:
    $$ \mathbf{n} = \frac{\mathbf{g}}{\|\mathbf{g}\|} $$
    [S2, S32, unofficial].

**Formulas:**
The central difference approximation for the gradient is:
$$ \nabla f(\mathbf{p}) \approx \begin{bmatrix} f(x+\epsilon, y, z) - f(x-\epsilon, y, z) \\ f(x, y+\epsilon, z) - f(x, y-\epsilon, z) \\ f(x, y, z+\epsilon) - f(x, y, z-\epsilon) \end{bmatrix} $$
[S30, S32, unofficial].
The normal is:
$$ \mathbf{n} = \text{normalize}(\nabla f(\mathbf{p})) $$
[S16, S35, unofficial].

**Parameters:**
*   **$\epsilon$ (Step Size):** Controls the trade-off between approximation accuracy and numerical stability. Too large $\epsilon$ causes aliasing and inaccurate normals on high-frequency details; too small $\epsilon$ leads to floating-point noise [S32, S59, unofficial]. Typical values: $0.0001$ to $0.001$ [S12, S27, S34, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** If $\|\mathbf{g}\| \approx 0$ (e.g., at singularities or flat regions with insufficient precision), normalization fails. A guard should return a default normal (e.g., $(0,0,1)$) or the view direction [S17, unofficial].
*   **Precision:** In double-precision contexts, $\epsilon$ can be smaller. In single-precision, $\epsilon$ must be large enough to exceed machine epsilon relative to the SDF values [S32, S59, unofficial].
*   **Sharp Edges:** Finite differences on sharp edges (where the SDF is not differentiable) produce averaged normals, which may appear soft or incorrect compared to analytic normals [S44].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_sdf_normal(p: Vector3, sdf_func, epsilon: float) -> Vector3:
    """
    Estimates the surface normal at point p using central finite differences.
    
    Args:
        p: The query point (Vector3).
        sdf_func: The signed distance function f(p).
        epsilon: The step size for finite differences.
        
    Returns:
        The normalized gradient vector.
    """
    # Define offset vectors
    offset_x = Vector3(epsilon, 0.0, 0.0)
    offset_y = Vector3(0.0, epsilon, 0.0)
    offset_z = Vector3(0.0, 0.0, epsilon)
    
    # Evaluate SDF at perturbed positions
    # Note: Division by 2*epsilon is omitted as it cancels during normalization
    grad_x = sdf_func(p + offset_x) - sdf_func(p - offset_x)
    grad_y = sdf_func(p + offset_y) - sdf_func(p - offset_y)
    grad_z = sdf_func(p + offset_z) - sdf_func(p - offset_z)
    
    # Construct gradient vector
    gradient = Vector3(grad_x, grad_y, grad_z)
    
    # Handle zero-length gradient (degenerate case)
    length = gradient.length()
    if length < 1e-6:
        return Vector3(0.0, 0.0, 1.0) # Default normal
        
    # Normalize and return
    return gradient / length
```

## Subtopics

### [Lambertian Shading with Gradient Normals](1229-lambertian-shading-with-gradient-normals.md)
Lambertian Shading with Gradient Normals applies the Lambertian reflectance model using the normals estimated via finite differences. It is chosen when diffuse lighting is required for implicit surfaces without explicit meshes. It differs from sibling methods by relying on the numerical gradient of the scalar field rather than analytic normals or tetrahedral interpolation.

### [Tetrahedral Normal Estimation](1986-tetrahedral-normal-estimation.md)
Tetrahedral Normal Estimation approximates the normal by evaluating the SDF at four vertices of a tetrahedron inscribed in the voxel cube, reducing SDF queries from six to four. It is distinguished by its reliance on geometric symmetry rather than axis-aligned offsets. It is chosen when performance is critical and slight accuracy trade-offs are acceptable.

### [Face Normal Computation](2042-face-normal-computation.md)
Face Normal Computation determines the geometric orientation of a polygonal face or voxel grid cell to provide a surface normal. It is used when explicit vertex normals are unavailable or a flat geometric normal is required for primary-ray shading in voxel/mesh-based rendering.

### [Normal Regularization](3704-normal-regularization.md)
Normal Regularization alleviates numerical instability (jitter) of surface normals near the UDF zero-level set by replacing naively sampled normals with interpolated normals from the neighborhood. It prevents high-contrast artifacts in 2D images. It is chosen when finite difference normals exhibit excessive noise or jitter.

### [Gradient-Based Surface Normal Estimation](4261-gradient-based-surface-normal-estimation.md)
Gradient-Based Surface Normal Estimation computes normals for volumetric representations by evaluating the spatial gradient of a local density field. It derives normals "on the fly" from the scalar field, ensuring coherence with voxel resolution. It is distinct from Scene-Space Normal Estimation by operating in volume space and from Tetrahedral Normal Estimation by using axis-aligned differences.

### [Principal Curvature Direction Estimation](6492-principal-curvature-direction-estimation.md)
Principal Curvature Direction Estimation computes the principal directions of curvature for a 3D scalar field by solving for an angle $\theta$ in a local orthonormal basis. It provides directional curvature information rather than just surface normals.

### [Scene-Space Normal Estimation](10497-scene-space-normal-estimation.md)
Scene-Space Normal Estimation uses ray-casting and binary segmentation to determine object boundary locations and estimate normals in scene space. It operates directly on scene data structures, making it suitable for pipelines utilizing scene space normals.

### SDF Normal Estimation via Backpropagation
SDF Normal Estimation via Backpropagation estimates mesh vertex normals by computing the negative gradient of the occupancy output with respect to the vertex positions using automatic differentiation [S60, unofficial]. It is used in differentiable rendering pipelines where analytical gradients are unavailable or too complex [S60, unofficial].

### Normal Estimation based on 4D Linear Regression
Normal Estimation based on 4D Linear Regression integrates filtering and normal computation into one process to provide a smooth approximated gradient function, compensating for staircase artifacts caused by low sampling rates in shear-warp projection [S61, unofficial]. It is used in volume rendering to smooth normals [S61, unofficial].

### Surface Normal Computation from MDF
Surface Normal Computation from MDF computes surface normals for a point $p$ and closest point $v$ using the gradient of the distance function $d(p, v)$, ensuring the normal points away from the surface [S62]. It is used in Medial Axis Distance Fields [S62].

### Object Space Normal Estimation
Object Space Normal Estimation estimates normals based on the local geometry of the surface in object space, rather than scene density [S57]. It is primarily used for rendering binary volumes by quantizing density levels and estimating normals for each isodensity surface [S57].

## Operating characteristics
**Time Complexity:** $O(1)$ per hit point, requiring exactly 6 SDF evaluations [S1][S12, S33, unofficial].
**Memory Complexity:** $O(1)$, storing only the gradient vector and temporary SDF values [S32, unofficial].
**Performance:** The method is computationally inexpensive but requires 6 texture/function lookups per shading point. It is significantly faster than higher-order finite difference stencils (e.g., 5-point stencil requiring 5-9 samples) [S6].
**Strengths:** Simple to implement; works with any SDF definition (analytic, procedural, neural); provides smooth normals for smooth SDFs [S16, S33, unofficial].
**Limitations:** Accuracy degrades with large $\epsilon$ (aliasing) or small $\epsilon$ (noise); inaccurate on sharp edges or discontinuities; requires the SDF to be differentiable at the query point [S32, unofficial][S44][S59, unofficial].
**When to use:** Standard shading for smooth SDFs where analytic normals are unavailable [S33, S40, unofficial].
**When not to use:** When sharp, crisp edges are required (use analytic normals or [Tetrahedral Normal Estimation](1986-tetrahedral-normal-estimation.md) with care); when performance is extremely constrained and 6 samples are prohibitive (use [Tetrahedral Normal Estimation](1986-tetrahedral-normal-estimation.md) with 4 samples) [S2, S4, unofficial].

## Background
The method originates from the fundamental property that the gradient of a distance function is orthogonal to its level sets (isosurfaces) [S5, S30, unofficial]. It was popularized in the context of ray marching and SDFs by Inigo Quilez and others in the demoscene and real-time graphics community [S5, S16, unofficial]. It is a standard technique in [Distance-field ray marching](21-distance-field-ray-marching.md) for shading implicit geometry [S1][S33, unofficial].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
[S3] Show HN: Volume rendering 3D data in Three.js and GLSL, https://github.com/SuboptimalEng/volume-rendering, fetched 2026-10-26 (unofficial)
[S4] Delyc/rayMarched-volumetric-rendering, https://github.com/Delyc/rayMarched-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S5] Show HN: Iridescent crystal with raymarching and signed distance fields, https://varun.ca/ray-march-sdf/, fetched 2026-10-26 (unofficial)
[S6] Self-Supervised Dual Contouring, https://inria.hal.science/hal-04590906v1, fetched 2026-10-26
[S7] shalu-007-git/isosurface-ray-casting, https://github.com/shalu-007-git/isosurface-ray-casting, fetched 2026-10-26 (unofficial)
[S8] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S9] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S10] a1k0n/asciitracing, https://github.com/a1k0n/asciitracing, fetched 2026-10-26 (unofficial)
[S11] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S12] AlerianEmperor/SDF-Sphere-Tracing, https://github.com/AlerianEmperor/SDF-Sphere-Tracing, fetched 2026-10-26 (unofficial)
[S13] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S14] AustinGoodman/non-linear-sphere-tracing, https://github.com/AustinGoodman/non-linear-sphere-tracing, fetched 2026-10-26 (unofficial)
[S15] BorisTheBrave/mc-dc, https://github.com/BorisTheBrave/mc-dc, fetched 2026-10-26 (unofficial)
[S16] Ray Marching and Signed Distance Functions, http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/, fetched 2026-10-26 (unofficial)
[S17] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S18] vulekhanh/VolumeRendering, https://github.com/vulekhanh/VolumeRendering, fetched 2026-10-26 (unofficial)
[S19] Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, https://github.com/Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, fetched 2026-10-26 (unofficial)
[S20] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S21] Adam-Gleave/sdf-terrain, https://github.com/Adam-Gleave/sdf-terrain, fetched 2026-10-26 (unofficial)
[S22] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S23] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S24] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S25] ollituohenmaa/raymarching-pathtracer, https://github.com/ollituohenmaa/raymarching-pathtracer, fetched 2026-10-26 (unofficial)
[S26] ChadMcKinney/Dirac-Sea-Engine, https://github.com/ChadMcKinney/Dirac-Sea-Engine, fetched 2026-10-26 (unofficial)
[S27] arguskern/lumina-graphics-lab, https://github.com/arguskern/lumina-graphics-lab, fetched 2026-10-26 (unofficial)
[S28] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S29] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
[S30] Ray Marching and Signed Distance Functions, https://github.com/guevara/read-it-later/issues/4535, fetched 2026-10-26 (unofficial)
[S31] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S32] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S33] Ray Marching: Signed Distance Fields, Sphere Tracing, and SDF Shading, https://tutorials.tektite.studio/ray-marching/, fetched 2026-10-27 (unofficial)
[S34] markpmlim/RaymarchingAndSDF, https://github.com/markpmlim/RaymarchingAndSDF, fetched 2026-10-27 (unofficial)
[S35] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
[S36] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S37] AJKKinney/Mandelbulb, https://github.com/AJKKinney/Mandelbulb, fetched 2026-10-27 (unofficial)
[S38] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
[S39] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S40] An Introduction to Raymarching - Maxime Garcia, https://typhomnt.github.io/teaching/ray_tracing/raymarching_intro/, fetched 2026-10-27 (unofficial)
[S41] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S42] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
[S43] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S44] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S45] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S46] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
[S47] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
[S48] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
[S49] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
[S50] Ray Marching with Signed Distance Fields - Scribd, https://www.scribd.com/document/831412915/1-Ray-Marching-and-Signed-Distance-Fields, fetched 2026-10-27 (unofficial)
[S51] Sphere Tracing and Ray Marching, https://deepwiki.com/ronja-tutorials/ShaderTutorials/5-sphere-tracing-and-ray-marching, fetched 2026-10-27 (unofficial)
[S52] GPU Ray Marching Techniques, https://www.scribd.com/document/814069988/5-GPU-Ray-Marching, fetched 2026-10-28 (unofficial)
[S53] Ray Marching with Three.js Guide, https://www.scribd.com/document/882338593/Ray-Marching-in-Three-js, fetched 2026-10-28 (unofficial)
[S54] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S55] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S56] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S57] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
[S58] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
[S59] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
[S60] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S61] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S62] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
