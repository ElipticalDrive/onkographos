# SDF Transformation

Also called: SDF Transforms, Transformed SDF, Transforming SDFs, SDF Affine Transformation, SDF Model Transformations, SDF Inverse Transformation, SDF Spatial Transformations, Rigid Body Transformation of SDFs, Inverse Transformation for Rotation/Translation

## Overview
SDF Transformation is a technique for applying geometric transformations (rotation, translation, scaling, and deformation) to Signed Distance Field primitives by manipulating the query point coordinates before evaluating the distance function [S1, S2, S3, S8, unofficial]. It solves the problem of positioning, orienting, and scaling implicit surfaces without modifying the underlying primitive definitions or requiring mesh regeneration [S3, S5, S9, unofficial]. Unlike [Spatial and nD transforms](3106-spatial-and-nd-transforms.md) which describe the general coordinate change, this method specifically details the inverse-application strategy and the necessary distance compensation for non-isometric transforms [S1, S3, S10, unofficial]. It sets itself apart from [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md) by handling general affine and non-linear deformations, though uniform scaling is a specific sub-case requiring distance multiplication [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A query point $\mathbf{p}$ in world space, a transformation $T$ (defined by parameters such as translation vector $\mathbf{d}$, rotation matrix $R$, scale factor $s$, or deformation function), and a base SDF function $f(\mathbf{x})$ [S3, S8, S10, unofficial].
- Output: The signed distance $d$ from $\mathbf{p}$ to the transformed surface, and optionally the surface normal $\mathbf{n}$ [S3, S9, unofficial].

**Data Structures:**
- Transformation Matrix: A $4 \times 4$ homogeneous matrix $M$ representing the local-to-world transform (Scale, Rotation, Translation) [S11, unofficial].
- Inverse Matrix: $M^{-1}$, used to transform the query point from world space to the primitive's local space [S3, S11, unofficial].
- Parameters:
  - Translation $\mathbf{d}$: 3D vector [S5, S8, unofficial].
  - Rotation: Angle $\theta$ and axis (e.g., Y-axis) [S8, S11, unofficial].
  - Scale $s$: Scalar (uniform) or vector (non-uniform) [S8, S10, unofficial].
  - Deformation: Coefficients for bending, twisting, or warping [S8, S14, unofficial].

**Steps:**
1. **Compute Inverse Transform:** Determine the inverse of the transformation $T$ that maps world coordinates to the primitive's local coordinates. For rigid body transformations (rotation/translation), this is the inverse matrix $T^{-1}$ [S3, S10, S12, unofficial].
2. **Transform Query Point:** Apply $T^{-1}$ to the world-space query point $\mathbf{p}$ to obtain the local-space point $\mathbf{p}_{local}$ [S3, S10, S12, unofficial].
   - For affine transforms: $\mathbf{p}_{local} = T^{-1}(\mathbf{p})$ [S10, unofficial].
   - For specific operations like translation by $\mathbf{d}$: $\mathbf{p}_{local} = \mathbf{p} - \mathbf{d}$ [S8, unofficial].
   - For rotation by angle $\theta$ around Y-axis: $\mathbf{p}_{local} = \text{rotateY}(-\theta) \cdot \mathbf{p}$ (using transpose for inverse in GLSL) [S3, S6, unofficial].
3. **Evaluate Base SDF:** Query the base distance function with the transformed point: $d' = f(\mathbf{p}_{local})$ [S3, S10, S12, unofficial].
4. **Compensate Distance (Non-Isometric):** If the transformation is not an isometry (e.g., scaling or non-linear deformation), adjust the returned distance $d'$ to preserve the signed distance property [S1, S10, S12, unofficial].
   - For uniform scale $s$: $d = s \cdot d'$ [S3, S10, S12, unofficial][S13].
   - For non-linear deformations: Use Lipschitz bounds or accept that the result is an approximation [S1, unofficial][S13].
5. **Return Result:** Return the compensated distance $d$ [S3, S10, unofficial].

**Formulas:**
- General Transformed Surface: $f(T^{-1}(\mathbf{x})) = 0$ [S10, S12, unofficial].
- Isometry Distance Preservation: $d(\mathbf{x}, I \circ f^{-1}(0)) = d(I^{-1}(\mathbf{x}), f^{-1}(0))$ [S10, S12, unofficial]. No distance adjustment needed.
- Uniform Scale: $S(\mathbf{x}) = s\mathbf{x}$. Inverse: $S^{-1}(\mathbf{x}) = \frac{1}{s}\mathbf{x}$. Distance: $d(\mathbf{x}, S(f^{-1}(0))) = s \cdot d(S^{-1}(\mathbf{x}), f^{-1}(0))$ [S10, S12, unofficial][S13].
- Twist: $\mathbf{q} = \text{rotateY}(\mathbf{p}, 0.5 \times p_y)$ [S1, S8, unofficial].
- Bend: Rotation in X-Y plane by angle proportional to X [S2, S8, unofficial].
- Repetition (Infinite Tile): $\mathbf{p}' = \text{mod}(\mathbf{p} + 0.5c, c) - 0.5c$ [S2, S8, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transform_sdf(p: Vector3, transform: Transform, base_sdf: Callable[[Vector3], float]) -> float:
    """
    Evaluate an SDF with a geometric transformation applied.
    
    Args:
        p: Query point in world space.
        transform: Object containing transformation parameters (matrix, scale, etc.).
        base_sdf: The underlying SDF function (e.g., sphere, box).
        
    Returns:
        Signed distance to the transformed surface.
    """
    # Step 1 & 2: Transform point to local space
    # For affine transforms (Rotation, Translation, Uniform Scale)
    if transform.is_affine:
        # Compute inverse transform matrix
        # Note: In GLSL, inverse of rotation is transpose (negative angle)
        inv_matrix = transform.get_inverse_matrix()
        
        # Convert point to homogeneous coordinates
        p_homogeneous = Vector4(p.x, p.y, p.z, 1.0)
        
        # Apply inverse transform
        p_local_homogeneous = inv_matrix * p_homogeneous
        
        # Extract xyz
        p_local = Vector3(p_local_homogeneous.x, p_local_homogeneous.y, p_local_homogeneous.z)
    else:
        # For non-linear deformations (Twist, Bend, Warp)
        p_local = transform.apply_inverse(p)

    # Step 3: Evaluate base SDF
    d_local = base_sdf(p_local)

    # Step 4: Compensate distance for non-isometric transforms
    if transform.is_scale:
        # Uniform scaling requires multiplying distance by scale factor
        # d_new = s * d_old
        d_compensated = transform.scale_factor * d_local
    elif transform.is_deformation:
        # Non-linear deformations break strict SDF properties.
        # For accurate bounds, Lipschitz constants are required [S1, unofficial][S13].
        # Here we return the raw value, noting it may not be a true distance.
        d_compensated = d_local
    else:
        # Isometries (Rotation, Translation) preserve distance
        d_compensated = d_local

    return d_compensated

def get_gradient(p: Vector3, transform: Transform, base_sdf: Callable[[Vector3], float]) -> Vector3:
    """
    Compute the gradient (normal) for a transformed SDF.
    Requires transforming the gradient vector back to world space.
    """
    # Transform point to local space
    if transform.is_affine:
        inv_matrix = transform.get_inverse_matrix()
        p_homogeneous = Vector4(p.x, p.y, p.z, 1.0)
        p_local = Vector3((inv_matrix * p_homogeneous).xyz)
    else:
        p_local = transform.apply_inverse(p)
        
    # Get local gradient
    grad_local = base_sdf.gradient(p_local)
    
    # Transform gradient to world space
    # The gradient transforms with the inverse transpose of the position transform
    # For rigid body transforms, this is just the rotation part of the forward matrix
    if transform.is_affine:
        world_matrix = transform.get_forward_matrix()
        # Extract rotation part (upper 3x3)
        rot_matrix = world_matrix.get_rotation_matrix()
        grad_world = rot_matrix * grad_local
    else:
        # For non-linear, Jacobian is needed [S13]
        grad_world = grad_local # Approximation
        
    return grad_world
```

**Edge Cases and Pitfalls:**
- **GLSL Matrix Inversion:** WebGL/GLSL lacks a built-in matrix inversion routine. For rotation matrices, use the transpose (negative angle) as the inverse [S3, S6, unofficial].
- **Non-Uniform Scaling:** Non-uniform scaling distorts distances anisotropically. Simple multiplication by a scalar is insufficient; conservative bounds or Lipschitz constants are required to prevent ray tunneling [S1, unofficial][S13].
- **Non-Linear Deformation:** Deformations like twist or bend modify space as a function of position, breaking local linearity. This can invalidate normal estimation and step sizes in ray marching [S11, unofficial][S13].
- **Repetition Boundaries:** Infinite tiling requires the position to be in the positive quadrant for modulo operations to work correctly, otherwise the shape may disappear [S8, unofficial].

## Subtopics

### [Lipschitz Transformation Bounds](1557-lipschitz-transformation-bounds.md)
Lipschitz Transformation Bounds provide a method for computing accurate distance fields for implicitly defined surfaces after applying spatial transformations (such as scaling, rotation, or deformation) by adjusting the distance estimate using the Lipschitz constant of the inverse transformation. Unlike standard Euclidean ray marching which assumes the metric is preserved, this method compensates for the distortion introduced by non-isometric transforms, preventing ray tunneling and ensuring the distance function remains a valid lower bound. It sets itself apart from [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md) by handling general non-uniform and non-linear deformations, and from [Spatial and nD transforms](3106-spatial-and-nd-transforms.md) by explicitly deriving and applying the Lipschitz bound to correct the distance value rather than just transforming the query point.

### [Implicit Surface Field Function Evaluation](1599-implicit-surface-field-function-evaluation.md)
Implicit Surface Field Function Evaluation computes the signed distance (or density) value $d = f(\mathbf{p})$ for a 3D point $\mathbf{p}$ relative to a target surface. It solves the problem of determining whether a point lies inside, outside, or on the boundary of a geometric primitive or voxelized field, which is the fundamental query required for ray marching, mesh extraction, and differentiable rendering. It differs from sibling methods like [Batched SDF Inference](2329-batched-sdf-inference.md) by focusing on the mathematical evaluation of a single or small set of points rather than large-scale batched inference, and from [SDF Validity Check with Relaxation](2399-sdf-validity-check-with-relaxation.md) by providing the raw metric value rather than a binary validity classification.

### [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md)
Uniform scaling of Signed Distance Fields (SDFs) is a technique to transform an implicit surface by a uniform factor while preserving the correctness of the distance value for ray marching. It solves the problem that scaling is not a rigid body transformation and distorts distances; without correction, the ray marching algorithm may overestimate the distance to the surface, causing the object to disappear. Unlike [Non-Uniform Scaling of SDFs](2423-non-uniform-scaling-of-sdfs.md), which requires conservative bounds to avoid overestimation, uniform scaling allows for an exact compensation of distance distortion.

### [Spatial and nD transforms](3106-spatial-and-nd-transforms.md)
Spatial and nD transforms allow the application of geometric transformations (rotation, translation, scaling, and permutation) to Signed Distance Field (SDF) primitives and discrete data structures by applying the inverse transformation to the query point rather than transforming the geometry itself. This approach ensures that the underlying data structures remain unmodified, preserving local linearity and allowing constant-time affine queries per dimension. It differs from direct geometry transformation by deferring the coordinate change to the evaluation step, which avoids distorting the stored data and supports non-uniform or spatially varying distortions.

### Deformation SDF
A technique to create a 'blob effect' by perturbing the position $\mathbf{p}$ with trigonometric functions before querying the SDF. This is noted as 'cheating' because it breaks the strict mathematical properties of the SDF [S14, unofficial].
- **Step:** Perturb coordinates:
  - $p_x += 0.02 \cos(8 p_y + 3 p_z)$
  - $p_y += 0.02 \sin(6 p_z + 7 p_x)$
  - $p_z += 0.02 \cos(8 p_x + 9 p_x)$ [S14, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query, assuming the transformation matrix inversion is precomputed. The cost is dominated by the base SDF evaluation and the matrix-vector multiplication [S3, S11, unofficial].
- **Memory Complexity:** $O(1)$ additional memory for storing the transformation matrix and parameters [S11, unofficial].
- **Performance:** Highly efficient on GPUs due to the lack of branching and simple arithmetic operations [S3, S11, unofficial].
- **Strengths:** No need to regenerate geometry; supports dynamic animations (e.g., time-varying rotation/translation) [S3, S11, unofficial].
- **Limitations:** Non-isometric transforms (scaling, deformation) require distance compensation or result in invalid SDFs (overestimation) [S1, S10, S11, unofficial]. Non-linear deformations break the guarantee of the distance field being a true metric, potentially causing ray marching artifacts [S11, unofficial][S13].

## Background
SDF transformations are a fundamental technique in ray marching and implicit surface rendering, popularized by shaders and demos in the demoscene and real-time graphics [S1, S3, S8, unofficial]. The mathematical foundation relies on the properties of isometries and the chain rule for distance functions [S10, S12, unofficial]. Key references include Hart's work on GPU ray marching of distance fields and Quilez's tutorials on SDF operations [S10, S11, S12, unofficial]. The technique is widely used in real-time rendering for procedural geometry, animation, and scene composition [S3, S8, unofficial].

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
