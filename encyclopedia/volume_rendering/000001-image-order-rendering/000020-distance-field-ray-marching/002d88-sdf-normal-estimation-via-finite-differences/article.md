# SDF Normal Estimation via Finite Differences

Also called: SDF Gradient Normal Estimation via Finite Differences, Normal Estimation via Finite Differences, Finite Difference Normal Estimation, SDF Gradient, SDF Gradient Normal, Inigo Quilez's Normal Method

## Overview
This method estimates the surface normal of an implicit surface defined by a Signed Distance Function (SDF) by numerically computing the gradient of the SDF at the intersection point [S2, S4, S5, unofficial]. It solves the problem of shading ray-marched geometry that lacks explicit mesh normals, allowing for realistic lighting without polygonization artifacts [S4, S5, unofficial]. Unlike sibling methods that may use analytic normals, tetrahedral interpolation, or regularization, this method relies on central finite differences of the scalar field $f$ at the hit point [S3, S14, unofficial]. It sets itself apart from [Gradient-Based Surface Normal Estimation](4261-gradient-based-surface-normal-estimation.md) by using explicit manual offset queries rather than a direct gradient sampling call or density kernel evaluation [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A point $\mathbf{p}$ on the surface (where $f(\mathbf{p}) \approx 0$), and the SDF function $f(\mathbf{p})$.
*   **Output:** A normalized surface normal vector $\mathbf{n}$.

**Data Structures:**
No specific data structures are required beyond the SDF function itself. The method operates on scalar values and 3D vectors.

**Parameters:**
*   $\epsilon$ (epsilon): The step size for finite differences. Typical values range from $0.0001$ to $0.01$ [S12, S15, S22, S27, S34, S40, S47, unofficial]. The magnitude affects the trade-off between approximation accuracy and numerical stability [S32, S45, S54, unofficial]. If $\epsilon$ is too small, floating-point noise dominates; if too large, the linear approximation error increases [S58, unofficial].

**Steps:**
1.  **Evaluate Central Point:** Compute the SDF value at the hit point $\mathbf{p}$ (optional for central differences, required for forward differences) [S22, S58, unofficial].
2.  **Sample Offsets:** Evaluate the SDF at points offset from $\mathbf{p}$ along the three coordinate axes ($x, y, z$) by $\pm \epsilon$ [S1][S33, S39, unofficial].
    *   For central differences, this requires 6 evaluations: $f(\mathbf{p} + \epsilon \hat{x})$, $f(\mathbf{p} - \epsilon \hat{x})$, $f(\mathbf{p} + \epsilon \hat{y})$, $f(\mathbf{p} - \epsilon \hat{y})$, $f(\mathbf{p} + \epsilon \hat{z})$, $f(\mathbf{p} - \epsilon \hat{z})$ [S1][S33, unofficial].
    *   For forward differences, this requires 3 evaluations: $f(\mathbf{p} + \epsilon \hat{x})$, $f(\mathbf{p} + \epsilon \hat{y})$, $f(\mathbf{p} + \epsilon \hat{z})$ [S49, unofficial].
3.  **Compute Gradient Components:** Calculate the partial derivatives using finite difference formulas [S7, S32, S39, unofficial].
    *   **Central Difference (Recommended):**
        $$ \frac{\partial f}{\partial x} \approx \frac{f(\mathbf{p} + \epsilon \hat{x}) - f(\mathbf{p} - \epsilon \hat{x})}{2\epsilon} $$
        $$ \frac{\partial f}{\partial y} \approx \frac{f(\mathbf{p} + \epsilon \hat{y}) - f(\mathbf{p} - \epsilon \hat{y})}{2\epsilon} $$
        $$ \frac{\partial f}{\partial z} \approx \frac{f(\mathbf{p} + \epsilon \hat{z}) - f(\mathbf{p} - \epsilon \hat{z})}{2\epsilon} $$
        The division by $2\epsilon$ is often omitted in implementation because it cancels out during normalization [S46, unofficial].
    *   **Forward Difference:**
        $$ \frac{\partial f}{\partial x} \approx \frac{f(\mathbf{p} + \epsilon \hat{x}) - f(\mathbf{p})}{\epsilon} $$
        (Similarly for $y$ and $z$) [S49, S57, unofficial].
4.  **Construct and Normalize Gradient:** Form the gradient vector $\nabla f = (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z})$ and normalize it to obtain the normal [S2, S45, S56, unofficial].
    $$ \mathbf{n} = \frac{\nabla f}{\|\nabla f\|} $$
    Note: The gradient points in the direction of greatest increase (outward from the surface). For shading, the normal is often used as-is, or negated depending on the coordinate system and lighting model convention [S35, S58, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_normal_central(p, sdf, epsilon):
    """
    Estimates the surface normal at point p using central finite differences.
    
    Args:
        p (vec3): The point on the surface.
        sdf (function): The Signed Distance Function.
        epsilon (float): The step size for finite differences.
        
    Returns:
        vec3: The normalized surface normal.
    """
    # Define offset vectors
    offset_x = vec3(epsilon, 0.0, 0.0)
    offset_y = vec3(0.0, epsilon, 0.0)
    offset_z = vec3(0.0, 0.0, epsilon)
    
    # Evaluate SDF at perturbed positions
    # Note: Division by 2*epsilon is omitted as it cancels during normalization
    grad_x = sdf(p + offset_x) - sdf(p - offset_x)
    grad_y = sdf(p + offset_y) - sdf(p - offset_y)
    grad_z = sdf(p + offset_z) - sdf(p - offset_z)
    
    # Construct gradient vector
    gradient = vec3(grad_x, grad_y, grad_z)
    
    # Normalize to get the normal
    # Handle degenerate case where gradient is zero (e.g., inside a flat volume or numerical error)
    if length(gradient) < 1e-6:
        return vec3(0.0, 1.0, 0.0) # Default normal or handle as error
        
    return normalize(gradient)

def estimate_normal_forward(p, sdf, epsilon):
    """
    Estimates the surface normal at point p using forward finite differences.
    Requires 3 fewer evaluations than central differences but is less accurate.
    """
    offset_x = vec3(epsilon, 0.0, 0.0)
    offset_y = vec3(0.0, epsilon, 0.0)
    offset_z = vec3(0.0, 0.0, epsilon)
    
    f_p = sdf(p)
    
    grad_x = sdf(p + offset_x) - f_p
    grad_y = sdf(p + offset_y) - f_p
    grad_z = sdf(p + offset_z) - f_p
    
    gradient = vec3(grad_x, grad_y, grad_z)
    
    if length(gradient) < 1e-6:
        return vec3(0.0, 1.0, 0.0)
        
    return normalize(gradient)
```

**Edge Cases and Pitfalls:**
*   **Sharp Edges/Cusps:** At sharp edges where the SDF is not differentiable, the finite difference approximation may produce inaccurate or jittery normals [S3704-normal-regularization.md].
*   **Numerical Stability:** If $\epsilon$ is too small relative to the precision of the SDF evaluation, the subtraction $f(\mathbf{p}+\epsilon) - f(\mathbf{p}-\epsilon)$ may suffer from catastrophic cancellation or be dominated by floating-point noise [S58, unofficial].
*   **Zero Gradient:** If the gradient magnitude is zero (e.g., at a local extremum or if the point is not on the surface), normalization fails. A fallback normal should be provided [S17, S32, unofficial].

## Subtopics

### [Lambertian Shading with Gradient Normals](1229-lambertian-shading-with-gradient-normals.md)
Lambertian Shading with Gradient Normals is a surface shading technique for ray-marched SDFs that approximates the surface normal using the numerical gradient of the field, then applies the Lambertian reflectance model to compute diffuse illumination. It solves the problem of shading implicit surfaces that lack explicit mesh normals, allowing for realistic diffuse lighting without polygonization artifacts. Unlike sibling methods that may use analytic normals, tetrahedral interpolation, or regularization, this method relies on finite-difference or analytic gradient estimation of the scalar field $f$ (or $\phi$) at the hit point.

### [Tetrahedral Normal Estimation](1986-tetrahedral-normal-estimation.md)
Tetrahedral Normal Estimation is a technique to approximate the surface normal of an SDF at a point $p$ using only four evaluations of the distance function, reducing computational cost by 33% compared to the standard six-call central difference method. It leverages the symmetry of a tetrahedron inscribed in the cube around $p$ to sum directional derivatives, providing accuracy comparable to central differences with fewer samples.

### [Face Normal Computation](2042-face-normal-computation.md)
Face normal computation determines the geometric orientation of a polygonal face or voxel grid cell to provide a surface normal for shading or ray intersection handling. It solves the problem of defining a consistent surface orientation when explicit vertex normals are unavailable or when a flat, geometric normal is required for primary-ray shading in voxel or mesh-based rendering.

### [Normal Regularization](3704-normal-regularization.md)
Normal Regularization is a technique to alleviate numerical instability (jitter) of surface normals near the Unsigned Distance Function (UDF) zero-level set (cusps) by replacing naively sampled normals with interpolated normals from the neighborhood. It prevents highly bright and dark regions in 2D images that occur with high-quality reconstruction without it.

### [Gradient-Based Surface Normal Estimation](4261-gradient-based-surface-normal-estimation.md)
This method computes surface normals for a voxelized volume by deriving the gradient of a locally computed density field on the fly, rather than storing explicit normal vectors or relying on mesh normals. It serves as a high-resolution, coherent normal estimation technique suitable for applications like particle collision response and ray marching, where memory efficiency and speed are prioritized over the global smoothness provided by large-support distance fields. It differs from SDF Normal Estimation via Finite Differences by using a direct gradient sampling call or a specific density kernel evaluation instead of manual finite difference offsets, reducing function call overhead and potential numerical instability.

### [SDF Bevel Generation](4312-sdf-bevel-generation.md)
SDF Bevel Generation is a technique for estimating 3D surface normals for lighting by sampling a Signed Distance Function (SDF) height field at four cardinal offsets, computing finite differences, and applying bevel shaping parameters to smooth sharp edges. It differs from standard gradient estimation by introducing explicit control over edge rounding and thickness, making it suitable for stylized or physically plausible beveled geometry in ray marching. It builds on SDF Normal Estimation via Finite Differences.

### [Principal Curvature Direction Estimation](6492-principal-curvature-direction-estimation.md)
Principal Curvature Direction Estimation computes the principal directions of curvature for a 3D scalar field (such as a Signed Distance Field) by solving for an angle $\theta$ in a local orthonormal basis, offering a faster alternative to extremal value quadratic forms. It sets itself apart from sibling methods like SDF Normal Estimation via Finite Differences by providing directional curvature information rather than just surface normals, and from Gradient-Based Surface Normal Estimation by explicitly handling the 2D subspace orthogonal to the gradient.

### SDF Normal Estimation via Backpropagation
Estimates mesh vertex normals by computing the negative gradient of the occupancy output with respect to the vertex positions [S59, unofficial]. It converts vertices to a PyTorch tensor, enables gradients, passes them through the model to get occupancy logits, computes the gradient by summing the output and calling backward, and extracts the normal as the negative gradient, normalized [S59, unofficial].(Note: This is a variant for differentiable rendering, not standard ray marching).

### Normal Estimation based on 4D Linear Regression
A normal estimation scheme that integrates filtering and normal computation into one process to provide a smooth approximated gradient function, compensating for staircase artifacts caused by low sampling rates in shear-warp projection [S60, unofficial]. It estimates surface inclination by investigating a close neighborhood of a given voxel, as exact surface normals are not available after discretization of continuous objects [S60, unofficial].

### Specular Highlight via Gradient Normal
Computes specular highlights by finding the volume surface intersection, estimating the surface normal via finite differences of the volume texture, and applying a Blinn-Phong shading model [S61, unofficial]. It ray marches to find surface intersection, computes normal N using central finite differences, and returns specular intensity as pow(dot(normalize(H),N), 2.0) [S61, unofficial].

### Surface Normal Computation from MDF
Computes surface normals $n(p, v)$ for a point $p$ and associated closest point $v$ using the gradient of the distance function $d(p, v)$, ensuring the normal points away from the surface [S62]. The formula is $n(p, v) = \frac{\nabla_p d(p, v)^T}{\|\nabla_p d(p, v)\|}$ [S62].

## Operating characteristics
**Time Complexity:** $O(1)$ per hit point, requiring a constant number of SDF evaluations (6 for central differences, 3 for forward differences) [S1][S49, unofficial].
**Memory Complexity:** $O(1)$, as no additional data structures are needed beyond the SDF function itself [S32, unofficial].
**Measured Performance:** The method is computationally inexpensive compared to analytic normal computation for complex primitives, but more expensive than pre-computed normals. It is widely used in real-time ray marching due to its simplicity and robustness [S2, S5, unofficial].
**Scaling:** Performance scales linearly with the number of ray hits. The cost is independent of scene complexity, as the SDF evaluation cost is the dominant factor, not the normal estimation [S34, unofficial].
**Strengths:**
*   Simple to implement [S16, S33, unofficial].
*   Works with any SDF, including those defined by complex Boolean operations or procedural functions where analytic normals are difficult or impossible to derive [S40, unofficial].
*   Provides smooth normals for smooth SDFs [S15, unofficial].
**Limitations:**
*   Approximate: Accuracy depends on the step size $\epsilon$ and the smoothness of the SDF [S32, S45, unofficial].
*   Inaccurate at sharp edges or cusps where the SDF is not differentiable [S3704-normal-regularization.md].
*   Requires multiple SDF evaluations, which can be a bottleneck if the SDF function is expensive to compute [S49, unofficial].
**When to use:** Use when analytic normals are unavailable or too complex to compute, and the SDF is smooth.
**When not to use:** Avoid for sharp-edged geometry where tetrahedral or regularization methods might be preferred, or when performance is critical and the SDF evaluation is very expensive (consider forward differences or tetrahedral methods).

## Background
The method originates from the mathematical property that the gradient of a scalar field is perpendicular to its level sets (isosurfaces) [S5, S30, unofficial]. It was popularized in the context of ray marching and signed distance fields by developers such as Inigo Quilez and Jamie Wong [S5, S16, unofficial]. It is a standard technique in real-time graphics for shading implicit surfaces [S2, S33, unofficial].

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
[S57] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
[S58] Ray Marching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
[S59] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S60] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S61] rbaravalle/webgl-volumetric, https://github.com/rbaravalle/webgl-volumetric, fetched 2026-10-27 (unofficial)
[S62] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
