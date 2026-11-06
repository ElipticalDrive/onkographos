# Direct Illumination Sampling with Mesh Intersection

Also called: Direct Integrator

## Overview
This method computes direct illumination for a surface point by sampling a light source and verifying visibility against explicit triangle meshes. It serves as the standard direct illumination integrator for polygonal scenes, distinguishing itself from SDF-based approaches by relying on geometric primitives rather than implicit distance fields. It builds on [Direct Light Sampling](3337-direct-light-sampling.md) by handling the specific geometry intersection requirements of mesh-based renderers.

## How it works
**Inputs:**
- A surface point $P$ and its normal $\mathbf{n}$ (from a primary ray intersection).
- A scene representation containing triangle meshes.
- A light source (emitter) definition.
- A BSDF (Bidirectional Scattering Distribution Function) defined at $P$.

**Outputs:**
- Accumulated direct illumination radiance $L$ at $P$.

**Data Structures:**
- `SurfaceInteraction3f`: A structure holding intersection details, including position, normal, UV coordinates, and primitive ID.
- `Vector3f`: 3D direction vectors.
- `Float`: Scalar values for distances and weights.

**Steps:**
1. **Sample Light Direction:** Sample a direction $\mathbf{l}$ towards the light source from $P$. Calculate the distance $d$ to the light. Define the light ray $\mathbf{r}_L = P + \mathbf{l} \cdot d$. [S1, unofficial]
2. **Evaluate BSDF and Light:** Evaluate the BSDF $\beta_r(\omega_i, \omega_o)$ at $P$ for the incoming direction $\mathbf{l}$ and the outgoing direction (from the camera/previous path). Evaluate the light's radiance $L_e(\mathbf{l})$. Compute the incoming radiance contribution $L_i = \beta_r \cdot L_e \cdot \frac{\cos \theta_l}{d^2}$ (assuming point/area light approximation as per standard direct lighting). [S1, unofficial]
3. **Shadow Ray Intersection:** Perform a ray intersection test against the scene meshes using $\mathbf{r}_L$. Let the result be `itx_shadow`. [S1, unofficial]
4. **Visibility Check:** Determine if the path is valid. The path is valid if `itx_shadow` is null (no occluder) or if `itx_shadow` hits the light source itself. [S1, unofficial]
5. **Accumulate:** If valid, add $L_i$ to the total radiance $L$. Otherwise, add zero. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_illumination_mesh(scene, surface_point, light, bsdf):
    """
    Compute direct illumination for a surface point using mesh intersection.
    
    Args:
        scene: Scene object with ray_intersect method.
        surface_point: Object containing position P, normal n, and BSDF.
        light: Light source object.
        bsdf: BSDF evaluator at surface_point.
        
    Returns:
        Float: Direct illumination radiance Li.
    """
    # Step 1: Sample light direction and distance
    # Note: Specific sampling strategy depends on light type (point, area, etc.)
    # This is a placeholder for the light sampling logic provided by the parent
    # Direct Light Sampling article.
    l_direction, distance_to_light = light.sample_direction(surface_point.position)
    
    # Construct the shadow ray
    # Offset P slightly along normal to avoid self-intersection
    P_offset = surface_point.position + surface_point.normal * 1e-4
    ray_L = Ray(origin=P_offset, direction=l_direction, max_t=distance_to_light)
    
    # Step 2: Evaluate BSDF and Light Radiance
    # beta_r is the BSDF value for incoming direction l_direction
    beta_r = bsdf.eval(surface_point.normal, l_direction)
    
    # Le is the radiance emitted by the light in direction l_direction
    Le = light.eval_radiance(l_direction)
    
    # Compute geometric attenuation and BSDF term
    # Assuming a point light or area light approximation where cos(theta) is handled
    # by the light sampling or BSDF evaluation. For a general area light, we might
    # need to account for the projected solid angle.
    # Here we assume the standard direct lighting formula:
    # Li = beta_r * Le * cos(theta_l) / d^2 * delta_term
    # However, the source notes simplify this to "evaluate BSDF to compute Li".
    # We will follow the source's high-level step.
    
    # Calculate Li based on source description: "Sample the emitter direction and 
    # evaluate the BSDF to compute incoming radiance Li"
    # This implies Li is computed here.
    cos_theta_l = max(0.0, dot(surface_point.normal, l_direction))
    
    # Avoid division by zero
    if distance_to_light < 1e-6:
        return 0.0
        
    Li = beta_r * Le * cos_theta_l / (distance_to_light ** 2)
    
    # Step 3: Check visibility using shadow ray
    itx_shadow = scene.ray_intersect(ray_L)
    
    # Step 4: Determine validity
    # Valid if no intersection (itx_shadow is None) or if it hits the light
    # Note: The source says "if both primary and secondary rays are valid" for mesh,
    # and "if the shadow ray is valid or hits the emitter" for SDF.
    # For mesh, we typically check if the shadow ray hit something closer than the light.
    # If itx_shadow is None, the path is clear.
    # If itx_shadow is not None, we check if the hit object is the light source.
    
    valid_ray = False
    if itx_shadow is None:
        valid_ray = True
    else:
        # Check if the intersection is with the light source
        # This requires checking the primitive ID or object ID against the light's ID
        if itx_shadow.primitive_id == light.primitive_id:
            valid_ray = True
            
    # Step 5: Accumulate
    # dr.select is a differentiable select operation
    # If valid_ray is true, return Li, else return 0.0 (black)
    if valid_ray:
        return Li
    else:
        return 0.0
```

**Edge Cases and Pitfalls:**
- **Self-Intersection:** The shadow ray origin must be offset from the surface point along the normal to prevent the ray from immediately hitting the surface it originated from. [G]
- **Precision:** Floating-point precision errors can cause shadow acne or false occlusions. Using a small epsilon offset is critical. [G]
- **Light Hit Detection:** Determining if a shadow ray hit the light source requires comparing the intersection's primitive ID or object ID with the light's ID. If the light is not a mesh primitive, this check may need to be distance-based (if `itx_shadow.t` is close to `distance_to_light`). [G]

## Subtopics

### SDF Direct Integrator
A direct illumination integrator for SDF-based scenes that performs ray intersection against the signed distance field rather than mesh geometry. It differs from the mesh integrator by using `sdf.ray_intersect` instead of `scene.ray_intersect`. The visibility check accumulates illumination if the shadow ray is valid or hits the emitter. [S1, unofficial]

## Operating characteristics
- **Time Complexity:** $O(N)$ per ray intersection, where $N$ is the number of triangles in the scene (or $O(\log N)$ with a BVH acceleration structure).
- **Memory Complexity:** $O(N)$ to store the scene geometry.
- **Strengths:** Accurate shadow casting for complex polygonal geometry; standard in production renderers.
- **Limitations:** Requires explicit mesh representation; slower than SDF intersection for simple scenes; sensitive to mesh quality and self-intersections.
- **When to use:** When rendering scenes with explicit triangle meshes and accurate shadows are required.
- **When not to use:** When using SDF-based rendering or when performance is critical and approximate shadows are acceptable.

## Background
The method is described in the context of differentiable rendering and neural scene optimization, specifically in the `zichenwang01/relaxed-boundary` repository. It is presented as a standard direct illumination integrator for mesh-based scenes. [S1, unofficial]

## Sources
[S1] zichenwang01/relaxed-boundary, https://github.com/zichenwang01/relaxed-boundary, fetched 2026-10-27 (unofficial)
