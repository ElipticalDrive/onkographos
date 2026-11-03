# SDF Normal Estimation via Finite Differences

Also called: SDF Normal Estimation, SDF Normal Calculation, Surface Normals from SDF, Central Difference Normal, Finite Difference Normals, Gradient Normal Estimation, Numerical Normal Computation, Gradient-Based Normal Estimation, Finite Difference Normal Estimation, Central Difference Normal Estimation, Finite Difference Normal Computation, Normal Approximation from Scalar Field, Central Differencing for Normal Estimation, Gradient Estimation via Finite Differences, Central finite difference normal estimation

## Overview
This method estimates the surface normal at a point on a Signed Distance Field (SDF) isosurface by numerically computing the gradient of the distance function using central finite differences along the three coordinate axes [S1, unofficial][S2, unofficial][S3, unofficial]. Unlike mesh-based rendering which stores explicit vertex normals, SDF ray marching [Distance-field ray marching](21-distance-field-ray-marching.md) relies on the field itself to provide shading information [S12, unofficial]. The gradient vector $\nabla f$ is perpendicular to the isosurface; normalizing this vector yields the surface normal $\mathbf{n}$ [S2, unofficial][S4, unofficial].

## How it works
**Inputs:**
- Query point $p = (p_x, p_y, p_z)$ on or near the isosurface, typically the hit point from ray marching [S1, unofficial][S3, unofficial].
- SDF evaluation function $f(p)$ returning the signed distance [S12, unofficial].

**Outputs:**
- Surface normal vector $\mathbf{n}$ [S2, unofficial].

**Data Structures:**
- No complex data structures required; the method requires access to the SDF function $f$ and a scalar $\epsilon$ [S3, unofficial].

**Parameters:**
- $\epsilon$: The perturbation step size. Sources cite specific values: $0.0005$ and $0.0007$ [S1, unofficial], $0.01$ [S3, unofficial][S4, unofficial], $0.05$ [S6, unofficial], $0.05\text{f}$ and $0.000001$ [S10, unofficial], and $0.0015$ [S13, unofficial]. The magnitude of $\epsilon$ directly affects the correctness of the normal calculation [S11, unofficial]. Smaller spacing causes noisy gradients, while larger spacing causes over-smoothed directions, requiring a trade-off [S12, unofficial].

**Algorithm Steps:**
1. Define the perturbation step size $\epsilon$ [S1, unofficial].
2. Evaluate the SDF at the six neighbors of $p$ along the coordinate axes:
   - $f_x^+ = f(p_x + \epsilon, p_y, p_z)$
   - $f_x^- = f(p_x - \epsilon, p_y, p_z)$
   - $f_y^+ = f(p_x, p_y + \epsilon, p_z)$
   - $f_y^- = f(p_x, p_y - \epsilon, p_z)$
   - $f_z^+ = f(p_x, p_y, p_z + \epsilon)$
   - $f_z^- = f(p_x, p_y, p_z - \epsilon)$.
3. Compute the gradient components using central differences:
   $$ \nabla f_x = \frac{f_x^+ - f_x^-}{2\epsilon} $$
   $$ \nabla f_y = \frac{f_y^+ - f_y^-}{2\epsilon} $$
   $$ \nabla f_z = \frac{f_z^+ - f_z^-}{2\epsilon} $$
   The gradient vector is $\mathbf{g} = (\nabla f_x, \nabla f_y, \nabla f_z)$ [S3, unofficial][S5, unofficial].
4. Normalize the gradient vector to obtain the surface normal:
   $$ \mathbf{n} = \frac{\mathbf{g}}{\|\mathbf{g}\|} $$
   [S2, unofficial][S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def estimate_sdf_normal(p, scene_sdf, epsilon=0.001):
    """
    Estimate the surface normal at point p using central finite differences.
    
    Args:
        p: tuple or list of 3 floats (x, y, z)
        scene_sdf: callable function f(p) -> float
        epsilon: step size for finite differences
        
    Returns:
        tuple: normalized normal vector (nx, ny, nz)
    """
    x, y, z = p
    
    # Evaluate SDF at neighbors along X axis
    f_x_plus = scene_sdf((x + epsilon, y, z))
    f_x_minus = scene_sdf((x - epsilon, y, z))
    
    # Evaluate SDF at neighbors along Y axis
    f_y_plus = scene_sdf((x, y + epsilon, z))
    f_y_minus = scene_sdf((x, y - epsilon, z))
    
    # Evaluate SDF at neighbors along Z axis
    f_z_plus = scene_sdf((x, y, z + epsilon))
    f_z_minus = scene_sdf((x, y, z - epsilon))
    
    # Compute gradient components
    grad_x = (f_x_plus - f_x_minus) / (2.0 * epsilon)
    grad_y = (f_y_plus - f_y_minus) / (2.0 * epsilon)
    grad_z = (f_z_plus - f_z_minus) / (2.0 * epsilon)
    
    # Normalize the gradient vector
    magnitude = math.sqrt(grad_x**2 + grad_y**2 + grad_z**2)
    
    # Guard against zero magnitude (degenerate case) [G]
    if magnitude < 1e-10:
        return (0.0, 0.0, 1.0)
        
    return (grad_x / magnitude, grad_y / magnitude, grad_z / magnitude)
```

**Edge Cases and Pitfalls:**
- **Precision:** Smaller spacing causes noisy gradients, while larger spacing causes over-smoothed directions, requiring a trade-off [S12, unofficial].
- **Loosely Spaced Voxel Grids:** In voxel-based SDF representations with large voxel sizes, the 6-sample central difference method may produce inaccurate normals. In such cases, using 18 or 26 samples provides a better estimate [S16, unofficial].

## Subtopics

### SDF Normal Estimation via Backpropagation
Estimates mesh vertex normals by computing the negative gradient of the occupancy output with respect to the vertex positions using automatic differentiation [S19, unofficial]. This method requires a differentiable model [S19, unofficial]. It involves converting vertices to tensors, passing them through the model to get occupancy logits, summing the output, and calling backward to extract the gradient [S19, unofficial]. The normal is the negative gradient, normalized [S19, unofficial].

## Operating characteristics
- **Time Complexity:** Requires 6 additional SDF evaluations per normal estimation [S2, unofficial]. For $N$ points, this is $6N$ function evaluations [S13, unofficial].
- **Accuracy:** Central differences provide a more accurate gradient estimate than forward or backward differences [S2, unofficial].
- **Strengths:** Simple to implement, works with any SDF [S4, unofficial].
- **Limitations:** Requires 6 extra samples, which can be costly in high-performance rendering if SDF evaluation is expensive [S2, unofficial]. Accuracy degrades with poor choice of $\epsilon$ or in loosely spaced voxel grids [S16, unofficial].
- **When to Use:** Standard SDF ray marching where analytical normals are unavailable and performance allows for 6 extra SDF queries [S4, unofficial].
- **When Not to Use:** When performance is critical and SDF evaluation is expensive, or when using voxel-based SDFs with large voxels where 18/26-sample methods are needed [S16, unofficial].

## Background
The technique of estimating surface normals via the gradient of a distance field is a fundamental concept in ray marching and implicit surface rendering [S4, unofficial]. It leverages the mathematical property that the gradient of a scalar field is perpendicular to its level sets (isosurfaces) [S2, unofficial]. The use of central finite differences for this purpose is a standard numerical analysis technique applied to SDFs [S3, unofficial]. Variants using more samples (18 or 26) are discussed in the context of voxel-based SDFs [S16, unofficial].

## Sources
[S1] Delyc/rayMarched-volumetric-rendering, https://github.com/Delyc/rayMarched-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] shalu-007-git/isosurface-ray-casting, https://github.com/shalu-007-git/isosurface-ray-casting, fetched 2026-10-26 (unofficial)
[S3] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S4] BorisTheBrave/mc-dc, https://github.com/BorisTheBrave/mc-dc, fetched 2026-10-26 (unofficial)
[S5] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S6] vulekhanh/VolumeRendering, https://github.com/vulekhanh/VolumeRendering, fetched 2026-10-26 (unofficial)
[S7] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S8] kushchevd/RayMarching, https://github.com/kushchevd/RayMarching, fetched 2026-10-26 (unofficial)
[S9] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S10] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
[S11] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S12] Ray Marching: Signed Distance Fields, Sphere Tracing, and SDF Shading, https://tutorials.tektite.studio/ray-marching/, fetched 2026-10-27 (unofficial)
[S13] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S14] An Introduction to Raymarching - Maxime Garcia, https://typhomnt.github.io/teaching/ray_tracing/raymarching_intro/, fetched 2026-10-27 (unofficial)
[S15] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S16] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S17] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S18] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
[S19] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
