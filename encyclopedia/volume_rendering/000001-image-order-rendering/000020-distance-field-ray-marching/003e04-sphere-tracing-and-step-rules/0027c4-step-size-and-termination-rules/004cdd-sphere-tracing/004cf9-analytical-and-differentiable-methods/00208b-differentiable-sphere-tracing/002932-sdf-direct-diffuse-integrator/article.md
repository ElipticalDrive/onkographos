# SDF Direct Diffuse Integrator

Also called: SDF Base Integrator, sdf-direct-diff

## Overview
The SDF Direct Diffuse Integrator is a differentiable rendering method that computes direct illumination for scenes defined by Signed Distance Functions (SDFs) [S1, unofficial]. It extends standard path tracing by incorporating a boundary integral term evaluated at grazing points, enabling accurate gradient computation for thin surfaces and silhouettes [S1, unofficial]. Unlike standard differentiable sphere tracing, it explicitly handles visibility discontinuities at object boundaries using a relaxed boundary formulation [S1, unofficial].

## How it works
The method computes the radiance $L$ for a camera ray by combining direct illumination from the primary intersection with a boundary correction term derived from grazing points [S1, unofficial].

### Inputs and Outputs
- **Input**: Camera ray $r$, scene definition (SDF volume, emitter, BSDF), sampling parameters, sensor configuration [S1, unofficial].
- **Output**: Radiance $L$ (RGB) and image plane gradients with respect to scene parameters [S1, unofficial].

### Data Structures
- **Interaction (`itx`)**: Stores position $p$, normal $n$, SDF value $d$, and emitter status at the primary intersection [S1, unofficial].
- **Grazing Point (`grz_i`)**: The location on the surface where the ray grazes the SDF isosurface [S1, unofficial].
- **Boolean (`is_grz_i`)**: Indicates if the primary intersection is a grazing point [S1, unofficial].

### Parameters
- `SDF_EPS`: Epsilon for sampling relaxed silhouette points. Value: $10^{-4}$ [S1, unofficial].
- `SDF_DERIV_EPS`: Epsilon for direction derivative sampling. Value: $10^{-1}$ [S1, unofficial].
- `ITX_EPS`: Intersection threshold. Value: $10^{-5}$ [S1, unofficial].
- `RAY_EPS`: Ray marching step size. Value: $10^{-3}$ [S1, unofficial].
- `NUM_GRZ`: Number of refinements for grazing point. Value: 12 [S1, unofficial].
- `GRZ_MODE`: Method for finding grazing points. Value: 'bisection' [S1, unofficial].

### Algorithm Steps
1. **Ray Intersection**: Perform ray intersection using the 'grz' method to obtain the primary intersection `itx`, a boolean `is_grz_i` indicating if the intersection is a grazing point, and the grazing point location `grz_i` [S1, unofficial].
   $$ \text{itx}, \text{is\_grz\_i}, \text{grz\_i} = \text{sdf.ray\_intersect}(\text{ray}, \text{method}='grz') $$
2. **Surface Emission**: Compute the surface emission $E$ at the intersection [S1, unofficial].
   $$ E = \text{itx.emitter}(\text{scene}=\text{scene}).\text{eval}(\text{itx}) $$
3. **Gradient Replacement Vector**: Calculate a velocity vector $v$ for gradient replacement based on the SDF value and normal $n$ [S1, unofficial].
   $$ v = -\frac{\text{sdf.at}(\text{itx}.p)}{\|n\|^2} n $$
4. **Position Adjustment**: Replace the intersection position `itx.p` with the velocity-adjusted position using gradient replacement [S1, unofficial].
   $$ \text{itx}.p = \text{replace\_grad}(\text{itx}.p, v) $$
5. **Incoming Radiance**: Sample the emitter direction and evaluate the BSDF to compute incoming radiance $L_i$ [S1, unofficial].
   $$ \text{itxL}, L_i = \text{scene.sample\_emitter\_direction}(\text{itx}, \text{samples}, \text{test\_visibility}=\text{False}) $$
   $$ L_i \leftarrow L_i \times \text{bsdf} $$
6. **Shadow Ray**: Check visibility for the direct illumination path using a shadow ray intersected against the SDF [S1, unofficial].
   $$ \text{itx\_shadow}, \text{is\_grz\_o}, \text{grz\_o} = \text{sdf.ray\_intersect}(\text{rayL}, \text{method}='grz') $$
7. **Grazing Point Retrieval**: Compute the grazing point `itx_grz` for the silhouette/boundary integral by retrieving the intersection at the grazing distance [S1, unofficial].
   $$ \text{itx\_grz} = \text{sdf.get\_intersect}(\text{ray}, t_{grz}, \text{is\_grz\_i}) $$
8. **Grazing Radiance**: Evaluate the radiance $L_{grz}$ at the grazing point by sampling the emitter and evaluating the BSDF at `itx_grz` [S1, unofficial].
9. **Boundary Integral**: Compute the boundary integral contribution $bp$ using the normal velocity $v_n = -\text{sdf.at}(\text{grz})$ and the difference in radiance between the grazing point and the primary intersection [S1, unofficial].
   $$ bp[\text{is\_grz\_i}] = v_n \frac{\text{detach}(L_{grz} - L)}{\text{SDF\_EPS}} $$
10. **Final Radiance**: Add the boundary integral to the final radiance using gradient replacement [S1, unofficial].
    $$ L = \text{replace\_grad}(L, L + bp) $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Differentiable primitives: replace_grad(x, v) sets gradient of x to v; detach(x) stops gradient flow;
# zeros_like(x) returns zero array of same shape; dot(a,b) computes dot product; squared_norm(x) = dot(x,x).

def sdf_direct_diffuse_integrator(ray, scene, sdf, samples, SDF_EPS):
    # Step 1: Ray Intersection with grazing detection
    itx, is_grz_i, grz_i = sdf.ray_intersect(ray, method='grz')
    
    # Step 2: Surface Emission
    E = itx.emitter(scene=scene).eval(itx)
    
    # Step 3: Calculate velocity vector for gradient replacement
    n = itx.normal
    d = sdf.at(itx.p)
    v = - (d / squared_norm(n)) * n
    
    # Step 4: Replace intersection position
    itx.p = replace_grad(itx.p, v)
    
    # Step 5: Sample emitter and evaluate BSDF
    itxL, Li = scene.sample_emitter_direction(itx, samples, test_visibility=False)
    Li = Li * itx.bsdf
    
    # Step 6: Shadow ray visibility check
    rayL = itxL.to_ray()
    itx_shadow, is_grz_o, grz_o = sdf.ray_intersect(rayL, method='grz')
    
    # Step 7: Get grazing point for boundary integral
    itx_grz = sdf.get_intersect(ray, grz_i.t, is_grz_i)
    
    # Step 8: Evaluate radiance at grazing point
    itxL_grz, L_grz = scene.sample_emitter_direction(itx_grz, samples, test_visibility=False)
    L_grz = L_grz * itx_grz.bsdf
    
    # Step 9: Compute boundary integral contribution
    vn = -sdf.at(grz_i)
    bp = zeros_like(Li)
    if is_grz_i:
        bp = vn * detach(L_grz - Li) / SDF_EPS
    
    # Step 10: Add boundary integral using gradient replacement
    L = replace_grad(Li, Li + bp)
    
    return L
```

### Edge Cases and Pitfalls
- **Division by Zero**: `SDF_EPS` is used as a divisor in the boundary integral term and must be strictly non-zero to prevent numerical errors [S1, unofficial].
- **Grazing Detection**: The 'grz' method must correctly identify grazing points; if `is_grz_i` is false, the boundary term is not applied [S1, unofficial].
- **Visibility**: The shadow ray visibility check uses `test_visibility=False` in the emitter sampling, but the shadow ray itself must be intersected to determine occlusion [S1, unofficial].

## Subtopics

### SDF Geometry Integrator
A direct illumination integrator for SDF-based scenes that uses a hardcoded two-sided diffuse BSDF with red reflectance [0.8, 0.2, 0.2] [S1, unofficial]. It performs ray intersection against the SDF, adjusts the normal direction to point against the ray direction, checks visibility using a shadow ray intersected against the SDF, and accumulates direct illumination if the shadow ray is valid or hits the emitter [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: Accurate gradients for silhouettes and thin surfaces; differentiable with respect to SDF parameters [S1, unofficial].
- **Limitations**: Requires a robust 'grz' intersection method; sensitive to `SDF_EPS` [S1, unofficial].
- **When to Use**: When rendering scenes with thin objects or when silhouette gradients are important for optimization [S1, unofficial].

## Background
The SDF Direct Diffuse Integrator is part of the "relaxed boundary" approach for differentiable rendering with SDFs [S1, unofficial]. It was introduced to address the limitations of standard differentiable sphere tracing, which fails to provide accurate gradients at object boundaries [S1, unofficial]. The method is implemented in the `zichenwang01/relaxed-boundary` repository [S1, unofficial].

## Sources
[S1] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
