# SDF Transformation

Also called: SDF Transforms, Transforming SDFs, Transforming SDF Geometry, Transformed SDF, SDF Affine Transformation, SDF Spatial Transformations, SDF Model Transformations, SDF Inverse Transformation, Inverse Transform SDF, Inverse Transformation for Rotation/Translation, Rigid Body Transformation of SDFs

## Overview
SDF Transformation is a technique for applying geometric transformations (rotation, translation, scaling, deformation) to Signed Distance Field primitives by manipulating the query point coordinates before evaluating the base distance function [S1, S3, S8, unofficial]. It solves the problem of positioning and animating implicit surfaces in world space without modifying the underlying primitive definitions [S3, S11, unofficial]. It sets itself apart from [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md) by handling general rigid body motions and non-linear deformations, and from [Implicit Surface Field Function Evaluation](1599-implicit-surface-field-function-evaluation.md) by explicitly addressing the coordinate mapping required to preserve geometric correctness [S1, S10, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A query point $\mathbf{x}$ in world space, and a transformation $T$ (defined by parameters such as translation vector $\mathbf{d}$, rotation matrix $R$, scale factor $s$, or deformation function).
- Output: A signed distance value $d$ relative to the transformed surface, and optionally the transformed point $\mathbf{x}'$ for further evaluation.

**Data Structures:**
- Transformation Matrix: A $4 \times 4$ homogeneous matrix $M$ representing the local-to-world transformation. For rigid bodies, this is typically composed of rotation $R$ and translation $\mathbf{t}$ [S11, unofficial].
- Parameters:
  - Translation $\mathbf{d}$: 3D vector [S5, S8, unofficial].
  - Rotation angle $\theta$: Scalar [S8, unofficial].
  - Scale $s$: Scalar (uniform) or vector (non-uniform) [S8, S10, unofficial].
  - Deformation coefficients: Scalars defining non-linear warps (e.g., twist rate) [S1, S8, unofficial].

**Method Steps:**
1. **Compute Inverse Transformation:** Determine the inverse of the desired transformation $T^{-1}$. For rigid body transformations (isometries), the inverse of a rotation by $\theta$ is a rotation by $-\theta$ [S3, S6, unofficial]. For translation by $\mathbf{d}$, the inverse is translation by $-\mathbf{d}$ [S8, unofficial]. For uniform scaling by $s$, the inverse is scaling by $1/s$ [S10, unofficial].
2. **Transform Query Point:** Apply $T^{-1}$ to the world-space query point $\mathbf{x}$ to obtain the local-space point $\mathbf{x}'$.
   - For matrix transformations: $\mathbf{x}' = (M^{-1} \cdot [\mathbf{x}, 1]^T)_{xyz}$ [S6, S11, unofficial].
   - For translation: $\mathbf{x}' = \mathbf{x} - \mathbf{d}$ [S8, unofficial].
   - For uniform scale: $\mathbf{x}' = \mathbf{x} / s$ [S10, unofficial].
   - For non-linear deformations (e.g., twist): Apply the inverse warp function to $\mathbf{x}$ [S1, S8, unofficial].
3. **Evaluate Base SDF:** Query the base signed distance function $f$ with the transformed point: $d' = f(\mathbf{x}')$ [S10, S12, unofficial].
4. **Adjust Distance (Non-Isometries):** If the transformation is not an isometry (e.g., scaling or non-linear deformation), adjust the returned distance $d'$ to preserve the metric property of the SDF [S10, S12, unofficial].
   - For uniform scaling by $s$: $d = s \cdot d'$ [S10, S12, unofficial].
   - For isometries (rotation, translation, reflection): No adjustment is needed; $d = d'$ [S10, S12, unofficial][S13].
   - For non-linear deformations: The distance value may no longer be a strict signed distance; step size bounds must be adjusted using Lipschitz constants [S1, unofficial][S13].

**Formulas:**
- General Transformed Surface: $f(T^{-1}(\mathbf{x})) = 0$ [S10, S12, unofficial].
- Isometry Distance Preservation: $d(\mathbf{x}, I \circ f^{-1}(0)) = d(I^{-1}(\mathbf{x}), f^{-1}(0))$ [S10, S12, unofficial].
- Uniform Scale Distance: $d(\mathbf{x}, S(f^{-1}(0))) = s \cdot d(S^{-1}(\mathbf{x}), f^{-1}(0))$ where $S^{-1}(\mathbf{x}) = \mathbf{x}/s$ [S10, S12, unofficial].
- Twist Transformation: $\mathbf{q} = \text{rotateY}(\mathbf{p}, k \cdot p_y)$, where $k$ is the twist factor [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transform_sdf(query_point: Vec3, transform_type: str, params: dict, base_sdf: callable) -> float:
    """
    Applies a geometric transformation to an SDF primitive.
    
    Args:
        query_point: World-space point to evaluate.
        transform_type: 'translation', 'rotation', 'scale', 'twist', 'matrix'.
        params: Dictionary containing transformation parameters.
        base_sdf: Function returning signed distance for the untransformed primitive.
        
    Returns:
        Signed distance to the transformed surface.
    """
    x = query_point
    
    # Step 1 & 2: Inverse Transform and Query Point
    if transform_type == 'translation':
        # Inverse of translation by d is -d
        d = params['translation']
        x_local = x - d
        
    elif transform_type == 'rotation':
        # Inverse of rotation by theta is -theta
        theta = params['angle']
        # Construct inverse rotation matrix (transpose for orthogonal matrices)
        c = cos(-theta)
        s = sin(-theta)
        # Assuming rotation around Y axis for this example
        rot_inv = Mat4([
            [c, 0, s, 0],
            [0, 1, 0, 0],
            [-s, 0, c, 0],
            [0, 0, 0, 1]
        ])
        x_homogeneous = Vec4(x, 1.0)
        x_local = (rot_inv * x_homogeneous).xyz
        
    elif transform_type == 'scale':
        # Inverse of uniform scale s is 1/s
        s = params['scale_factor']
        x_local = x / s
        
    elif transform_type == 'twist':
        # Inverse twist: rotate by negative angle proportional to y
        k = params['twist_factor']
        angle = -k * x.y
        c = cos(angle)
        s = sin(angle)
        # Rotate around Y axis
        x_local = Vec3(
            x.x * c - x.z * s,
            x.y,
            x.x * s + x.z * c
        )
        
    elif transform_type == 'matrix':
        # General inverse matrix multiplication
        M_inv = params['inverse_matrix']
        x_homogeneous = Vec4(x, 1.0)
        x_local = (M_inv * x_homogeneous).xyz
        
    else:
        raise ValueError("Unknown transform type")

    # Step 3: Evaluate Base SDF
    dist_local = base_sdf(x_local)
    
    # Step 4: Adjust Distance for Non-Isometries
    if transform_type == 'scale':
        s = params['scale_factor']
        dist = s * dist_local
    elif transform_type == 'twist':
        # Non-linear deformation breaks strict SDF property.
        # Distance is approximate; step size must be bounded by Lipschitz constant.
        # [G] For implementation, return raw distance but warn user to use 
        # Lipschitz bounds for ray marching step size.
        dist = dist_local
    else:
        # Isometries (translation, rotation, matrix rigid body) preserve distance
        dist = dist_local
        
    return dist
```

**Edge Cases and Pitfalls:**
- **GLSL Matrix Inversion:** WebGL/GLSL lacks a built-in matrix inversion routine. For rotation matrices, use the transpose (negative angle) instead of `invert()` [S3, S6, unofficial].
- **Non-Uniform Scaling:** Requires conservative bounds or Lipschitz adjustment to avoid overestimation of distance, which can cause the object to disappear during ray marching [S1, S3, S11, unofficial].
- **Deformation:** Non-linear spatial distortion (e.g., twist, bend) breaks local linearity, making normal estimation via finite differences inaccurate and requiring adjusted step sizes [S11, unofficial][S13].

## Subtopics

### [Lipschitz Transformation Bounds](1557-lipschitz-transformation-bounds.md)
Lipschitz Transformation Bounds compute the distances to implicitly defined surfaces after applying spatial transformations by adjusting the Lipschitz constant of the inverse transformation. This method provides a valid step size bound for ray marching when the underlying Signed Distance Field (SDF) is transformed [G]. It sets itself apart from [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md) by handling non-uniform, non-linear, and complex deformations (tapering, twisting) where a single scalar scale factor is insufficient, and from [Implicit Surface Field Function Evaluation](1599-implicit-surface-field-function-evaluation.md) by explicitly addressing the geometric distortion introduced by the coordinate transformation itself.

### [Implicit Surface Field Function Evaluation](1599-implicit-surface-field-function-evaluation.md)
Implicit Surface Field Function Evaluation computes the scalar value $d = f(x,y,z)$ for a query point in 3D space. It maps coordinates to a signed distance or density, where $d < 0$ indicates the interior, $d > 0$ the exterior, and $d = 0$ the surface contour.

### [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md)
Uniform scaling of Signed Distance Fields (SDFs) is a technique to transform an implicit surface by a uniform factor while preserving the correctness of the distance value for ray marching. It solves the problem that scaling is not a rigid body transformation and distorts distances; without correction, the ray marching algorithm may overestimate the distance to the surface, causing the object to disappear. Unlike [Non-Uniform Scaling of SDFs](2423-non-uniform-scaling-of-sdfs.md), which requires conservative bounds to avoid overestimation, uniform scaling allows for an exact compensation of distance distortion.

### [Spatial and nD transforms](3106-spatial-and-nd-transforms.md)
Spatial and nD transforms apply geometric transformations to Signed Distance Field (SDF) primitives by inverse-transforming the query point rather than the geometry itself. This approach allows for the rotation, translation, scaling, and permutation of SDF models within the distance function evaluation. It sets itself apart from direct geometry manipulation by maintaining the mathematical integrity of the distance field while enabling complex hierarchical or per-node transformations, particularly in multidimensional data structures.

### Deformation SDF
A technique to create a 'blob effect' by perturbing the position $p$ with trigonometric functions before querying the SDF. This is noted as 'cheating' because it breaks the strict mathematical properties of the SDF [S14, unofficial].
- **Step:** Perturb coordinates using trigonometric functions:
  - $p_x \leftarrow p_x + 0.02 \cos(8 p_y + 3 p_z)$
  - $p_y \leftarrow p_y + 0.02 \sin(6 p_z + 7 p_x)$
  - $p_z \leftarrow p_z + 0.02 \cos(8 p_x + 9 p_x)$ [S14, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, dominated by the cost of the base SDF evaluation and the transformation arithmetic.
- **Memory Complexity:** $O(1)$, requiring only storage for transformation parameters.
- **Strengths:** Enables complex scene composition, animation, and hierarchical modeling without modifying primitive definitions. Rigid body transformations preserve the SDF property exactly [S3, S10, unofficial].
- **Limitations:** Non-rigid transformations (scaling, deformation) break the strict SDF property, requiring heuristic adjustments to step sizes or normal estimation [S11, unofficial][S13]. Non-linear deformations can cause artifacts in normal calculation if finite differences are used [S11, unofficial].
- **When to Use:** Ideal for rigid body animation, camera-relative effects, and simple scaling. Use caution with non-linear deformations in contexts requiring precise normals or strict distance guarantees.

## Background
Originating from the foundational work on ray marching and signed distance fields in computer graphics, SDF transformations were popularized by blogs and tutorials such as those by Jamie Wong and 9bitScience [S1, S3, unofficial]. The mathematical formalization of implicit surface transformations, including the distinction between isometries and non-isometries, is detailed in academic works on GPU ray marching [S10, S12, unofficial]. The technique is widely used in real-time rendering, particularly in Shadertoy-style demos and procedural geometry generation [S1, S3, S4, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S3] Ray Marching and Signed Distance Functions, http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/, fetched 2026-10-26 (unofficial)
[S4] nopjia/WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
[S5] kyle-rosa/ray_marching, https://github.com/kyle-rosa/ray_marching, fetched 2026-10-26 (unofficial)
[S6] Ray Marching and Signed Distance Functions, https://github.com/guevara/read-it-later/issues/4535, fetched 2026-10-26 (unofficial)
[S7] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
[S8] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S9] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
[S10] GPU Ray Marching of Distance Fields, https://www2.compute.dtu.dk/pubdb/edoc/imm6392.pdf, fetched 2026-10-27 (unofficial)
[S11] GPU Ray Marching Techniques, https://www.scribd.com/document/814069988/5-GPU-Ray-Marching, fetched 2026-10-28 (unofficial)
[S12] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
[S13] Sphere tracing: a geometric method for the antialiased ray tracing of implicit surfaces, http://graphics.cs.illinois.edu/sites/default/files/zeno.pdf, fetched 2026-10-29
[S14] Introduction to Raymarching Distance Functions, https://observablehq.com/@gre/introduction-to-raymarching-distance-functions, fetched 2026-10-29 (unofficial)
