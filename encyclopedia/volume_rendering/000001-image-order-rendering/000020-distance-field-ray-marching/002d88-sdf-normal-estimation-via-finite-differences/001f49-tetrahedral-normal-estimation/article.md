# Tetrahedral Normal Estimation

Also called: 4-point normal, Quilez Normal Estimation, Tetrahedral Gradient Estimation, Tetrahedral SDF Normal Approximation, Tetrahedral gradient, Tetrahedron Gradient Normal Estimation, Tetrahedron Offset Normal Estimation

## Overview
Tetrahedral Normal Estimation is a technique to approximate the surface normal of a Signed Distance Field (SDF) at a point $p$ using only four evaluations of the distance function, reducing computational cost by 33% compared to the standard six-call central difference method [S1, unofficial]. It leverages the symmetry of a tetrahedron inscribed in the cube around $p$ to sum directional derivatives, providing accuracy comparable to central differences with fewer samples [S2, unofficial][S3].

## How it works
**Inputs:**
- Point $p \in \mathbb{R}^3$ where the normal is estimated.
- SDF function $map: \mathbb{R}^3 \to \mathbb{R}$.
- Step size $h$ (scalar).

**Outputs:**
- Normal vector $n \in \mathbb{R}^3$.

**Data Structures:**
- Offset vector $e = (e_x, e_y, e_z)$. Typically defined as $e = (h, -h, h)$ or derived from a base step size. The specific implementation in [S1, unofficial] uses a 2D vector construction `vec2(1.0, -1.0) * 0.0006` to generate 3D offsets.

**Steps:**
1. Define the four directional vectors $k_i$ corresponding to the vertices of a regular tetrahedron centered at the origin. Common choices for the directions (normalized) are:
   - $k_0 = (1, 1, 1)$
   - $k_1 = (1, -1, -1)$
   - $k_2 = (-1, 1, -1)$
   - $k_3 = (-1, -1, 1)$
   Note: These vectors are equidistant from each other and from the origin.
2. Evaluate the SDF at the four offset points:
   - $d_0 = map(p + h \cdot k_0)$
   - $d_1 = map(p + h \cdot k_1)$
   - $d_2 = map(p + h \cdot k_2)$
   - $d_3 = map(p + h \cdot k_3)$
3. Compute the weighted sum of the distances multiplied by their respective direction vectors. This approximates the gradient:
   $$ g = k_0 \cdot d_0 + k_1 \cdot d_1 + k_2 \cdot d_2 + k_3 \cdot d_3 $$
   *Note on [S1, unofficial]:* The specific pseudocode in [S1, unofficial] multiplies the returned distance by the *components* of the offset vector itself (e.g., `e.xyy * map(...)`). This is mathematically equivalent to scaling the direction vector by the step size if the direction vector components are $\pm 1$. Specifically, if $k_i$ has components $\pm 1$, then $h \cdot k_i$ has components $\pm h$. The formula in [S1, unofficial] `e.xyy * map(...)` implies the weight is the offset vector itself.
4. Normalize the result to obtain the normal:
   $$ n = \frac{g}{\|g\|} $$

**Formula:**
The general form is [S3]:
$$ normal(p) \approx normalize\left(\sum_{i=0}^{3} k_i map(p + h k_i)\right) $$
Where $k_i$ are the four directional vectors of the tetrahedron vertices, $h$ is the step size, and $map$ is the distance field function.

A specific implementation detail from [S1, unofficial] defines the offset vector $e$ and computes:
$$ n = normalize( e_{xyz}^{(0)} \cdot map(p + e^{(0)}) + e_{xyz}^{(1)} \cdot map(p + e^{(1)}) + e_{xyz}^{(2)} \cdot map(p + e^{(2)}) + e_{xyz}^{(3)} \cdot map(p + e^{(3)}) ) $$
Where the offsets $e^{(i)}$ are permutations of $(e_x, e_y, e_y)$, $(e_y, e_x, e_y)$, etc., derived from `vec2(1.0, -1.0)`.

**Parameters:**
- Step size $h$: [S1, unofficial] uses $0.0006$ derived from `vec2(1.0, -1.0) * 0.0006`. The optimal value depends on the scale of the SDF and the precision of the floating-point representation. Not given in [S2, unofficial][S3].

**Edge Cases and Pitfalls:**
- **Sharp Intersections:** The normal value is inaccurate at sharp intersections of multiple surfaces [S3]. This occurs because the SDF blends the distances, and the gradient sum does not correctly resolve the discontinuity.
- **Mitigation:** To avoid inaccuracies at intersections, evaluate only the SDF of the closest object using a mask based on material ID [S3].
- **Precision:** The method is less numerically accurate than central differences if the step size $h$ is not chosen carefully relative to the machine epsilon and the scale of the distance field values [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tetrahedral_normal(p, map_func, h):
    # Define the 4 tetrahedral directions (normalized is not strictly necessary 
    # if h scales them, but they must be equidistant)
    # Using the standard regular tetrahedron vertices scaled by h
    k0 = vec3( 1,  1,  1) * h
    k1 = vec3( 1, -1, -1) * h
    k2 = vec3(-1,  1, -1) * h
    k3 = vec3(-1, -1,  1) * h

    # Evaluate SDF at the 4 points
    d0 = map_func(p + k0)
    d1 = map_func(p + k1)
    d2 = map_func(p + k2)
    d3 = map_func(p + k3)

    # Sum the weighted distances
    # The weight is the direction vector itself (k_i)
    gradient = k0 * d0 + k1 * d1 + k2 * d2 + k3 * d3

    # Normalize to get the normal
    normal = normalize(gradient)
    
    return normal
```
*Note: The pseudocode above follows the general formula [S3]. The specific implementation in [S1, unofficial] uses a different construction of offsets from a 2D vector, but the principle of summing $k_i \cdot map(p+k_i)$ remains. If strict adherence to [S1, unofficial]'s specific offset generation is required:*
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tetrahedral_normal_pigjet(p, map_func, base_step):
    # From [S1, unofficial]: e = vec2(1.0, -1.0) * 0.0006
    # This generates 3D offsets by duplicating components
    # e.xyy, e.yyx, e.yxy, e.xxx
    # Let e_base = vec3(1.0, -1.0, 1.0) * base_step ? 
    # [S1, unofficial] says: e = vec2(1.0, -1.0) * 0.0006
    # Offsets: e.xyy, e.yyx, e.yxy, e.xxx
    # If e is vec2, how is it extended? 
    # Assuming e is treated as a 3D vector where the 3rd component is derived or e is actually vec3(1,-1,1) scaled?
    # [S1, unofficial] code snippet: e = vec2(1.0, -1.0) * 0.0006. 
    # Then: e.xyy, e.yyx, e.yxy, e.xxx. 
    # This implies e is a vec3 where x=1*step, y=-1*step, z=1*step? 
    # Or is it a vec2 used to generate 3D? 
    # "e.xyy" suggests e is a vec3 with components x, y, y. 
    # If e = vec3(1, -1, -1) * h, then e.xyy = (h, -h, -h). 
    # Let's assume the standard tetrahedral directions as per [S3] for clarity, 
    # as [S1, unofficial]'s specific vector construction is ambiguous without full code context.
    # However, [S1, unofficial] explicitly lists: e.xyy, e.yyx, e.yxy, e.xxx.
    # If we define e_vec = vec3(1, -1, 1) * h:
    # e_vec.xyy = (h, -h, -h) -> k1
    # e_vec.yyx = (-h, h, -h) -> k2
    # e_vec.yxy = (-h, -h, h) -> k3
    # e_vec.xxx = (h, h, h)   -> k0
    # This matches the standard tetrahedral directions.
    
    h = base_step
    e = vec3(1.0, -1.0, 1.0) * h
    
    d0 = map_func(p + e.xxx)
    d1 = map_func(p + e.xyy)
    d2 = map_func(p + e.yyx)
    d3 = map_func(p + e.yxy)
    
    gradient = e.xxx * d0 + e.xyy * d1 + e.yyx * d2 + e.yxy * d3
    return normalize(gradient)
```

## Subtopics

### [Lambertian Shading with Gradient Normals](1229-lambertian-shading-with-gradient-normals.md)
Uses gradient normals for diffuse shading.

### [Face Normal Computation](2042-face-normal-computation.md)
Computes normals from mesh faces, not SDFs.

### [Normal Regularization](3704-normal-regularization.md)
Smooths noisy normals.

### [Gradient-Based Surface Normal Estimation](4261-gradient-based-surface-normal-estimation.md)
General category including central differences.

### [SDF Bevel Generation](4312-sdf-bevel-generation.md)
Uses normals for beveling.

### [Principal Curvature Direction Estimation](6492-principal-curvature-direction-estimation.md)
Uses higher-order derivatives.

## Operating characteristics
- **Time Complexity:** 4 SDF evaluations per pixel/point. This is 33% faster than the 6-evaluation central difference method [S1, unofficial].
- **Memory Complexity:** Minimal, only storing the 4 distance values and the gradient vector.
- **Accuracy:** Provides accuracy comparable to central differences for smooth surfaces [S3]. However, it is less accurate at sharp intersections of multiple surfaces compared to methods that resolve the closest object explicitly [S3].
- **Strengths:** Reduced computational cost, good accuracy for smooth SDFs.
- **Limitations:** Inaccurate at sharp intersections; requires careful selection of step size $h$ to avoid numerical precision issues [S3].
- **When to use:** Real-time ray marching where performance is critical and surfaces are smooth.
- **When not to use:** Scenes with sharp intersections of multiple SDFs without masking, or where high precision is required at discontinuities.

## Background
The method is attributed to Inigo Quilez, a well-known graphics programmer and author of "Real-Time Rendering" shader tutorials [S2, unofficial]. It is widely used in real-time ray marching shaders for its efficiency. The technique is described in academic literature as a way to obtain surface normals with the same accuracy as central differences using only four evaluations [S3].

## Sources
[S1] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S2] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
[S3] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
