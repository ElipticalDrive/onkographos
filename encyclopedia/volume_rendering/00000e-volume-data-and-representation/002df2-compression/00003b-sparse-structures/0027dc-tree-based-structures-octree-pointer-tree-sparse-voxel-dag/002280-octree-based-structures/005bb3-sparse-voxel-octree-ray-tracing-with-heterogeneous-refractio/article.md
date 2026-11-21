# Sparse Voxel Octree Ray Tracing with Heterogeneous Refraction

Also called: Real-time GPU Ray Tracing with Volumetric Effects

## Overview
A real-time GPU ray tracing method that represents volumes as a sparse voxel octree and bends light through see-through objects with spatially varying indices of refraction [S1, unofficial]. It differs from standard SVO ray tracing by storing per-voxel optical properties (IOR, reflectivity) to enable heterogeneous refraction, where light rays are re-bent at every voxel boundary with a different index of refraction, rather than treating the volume as a homogeneous medium or refracting only at the surface [S1, unofficial].

## How it works
**Inputs:** A scene represented as a Sparse Voxel Octree (SVO) generated from meshes (e.g., via `TreeGen`), camera parameters, and light definitions. The SVO is built with a typical depth of ~8 for quick generation, supporting up to 2048³ resolution [S1, unofficial].

**Outputs:** A rendered image written directly to an OpenGL texture [S1, unofficial].

**Data Structures:**
*   **Sparse Voxel Octree:** Recursive subdivision where empty space is never stored. Nodes are traversed using the Laine & Karras algorithm [S1, unofficial].
*   **Voxel Attachment:** Stored at leaf nodes (or hit points). It packs four fields into **12 bytes**:
    1.  Colour (3 bytes, likely RGB)
    2.  Normal (3 bytes, likely RGB or quantized vector)
    3.  Reflectivity (1 byte)
    4.  Index of Refraction (IOR) (5 bytes, likely fixed-point or quantized float) [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Assign one CUDA thread per pixel [S1, unofficial].
2.  **Primary Ray Casting:** Cast the primary ray through the SVO using Laine & Karras traversal [S1, unofficial].
3.  **Hit Processing:** Upon hitting a voxel, retrieve the Voxel Attachment (colour, normal, reflectivity, IOR) [S1, unofficial].
4.  **Lighting Calculation:** Compute ambient, diffuse, and specular lighting based on the hit normal and material properties [S1, unofficial].
5.  **Shadow Rays:** For each light source, cast a shadow ray.
    *   **Attenuation:** Calculate point light attenuation using:
        $$ \text{attenuation} = \frac{1.0}{\text{light.constant\_att} + \text{light.linear\_att} \cdot d + \text{light.quadratic\_att} \cdot d^2} $$
        where $d$ is the light distance [S1, unofficial].
    *   **Spotlight Cone Check:** If the light is a spotlight, clamp the cosine of the angle between the light direction and the ray direction:
        $$ c = \max(0.0, \text{dot}(\text{light\_dir}, \text{normalize}(\text{light.direction}))) $$
        If $c < \cos(\text{light.cutoff})$, attenuation is 0. Otherwise, multiply attenuation by $c^{\text{light.exponent}}$ [S1, unofficial].
    *   **Self-Intersection Avoidance:** Offset the shadow ray origin to prevent self-intersection artifacts:
        $$ \text{origin} = \text{hit\_pos} + \text{direction} \cdot \sqrt{10} \cdot 2^{-\text{hit\_scale}} $$
        where $\text{hit\_scale}$ relates to the voxel size at the hit point [S1, unofficial].
6.  **Reflection (Optional):** If reflection is enabled, calculate the reflection vector:
    $$ \text{refl} = \text{norm} \cdot (2.0 \cdot \text{dot}(\text{dir}, \text{norm})) - \text{dir} $$
    The implementation limits recursive reflection bounces to zero (`MAX_REFLECTION_ITERATIONS = 0`) [S1, unofficial].
7.  **Heterogeneous Refraction:** If the voxel is see-through (IOR $\neq$ 1.0 or similar threshold):
    *   Bend the ray using Snell's law at the entry point.
    *   Step through the volume.
    *   At each voxel boundary, if the IOR changes, re-bend the ray using Snell's law with the new IOR [S1, unofficial].
8.  **Output:** Write the final color to the OpenGL texture [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray(ray_origin, ray_direction, octree, lights, camera):
    # Step 1: Traverse SVO
    hit = traverse_svo_laine_karras(octree, ray_origin, ray_direction)
    
    if hit is None:
        return background_color

    # Step 2: Retrieve Voxel Attachment (12 bytes)
    colour, normal, reflectivity, ior = hit.voxel_attachment
    
    # Step 3: Lighting
    color = ambient_light
    for light in lights:
        # Shadow Ray
        light_dir = normalize(light.position - hit.position)
        distance = length(light.position - hit.position)
        
        # Self-intersection offset
        outset = sqrt(10.0) * exp2(-hit.scale)
        shadow_origin = hit.position + light_dir * outset
        shadow_ray = Ray(shadow_origin, light_dir)
        
        if is_occluded(octree, shadow_ray):
            continue
            
        # Attenuation
        att = 1.0 / (light.constant + light.linear * distance + light.quadratic * distance * distance)
        
        # Spotlight check
        if light.is_spotlight:
            cos_angle = max(0.0, dot(light.direction, light_dir))
            if cos_angle < cos(light.cutoff):
                att = 0.0
            else:
                att *= pow(cos_angle, light.exponent)
                
        color += att * light.color * (diffuse + specular)

    # Step 4: Reflection (Limited to 0 bounces in this impl)
    # If reflection were enabled:
    # refl_dir = reflect(ray_direction, normal)
    # color += reflectivity * trace_ray(hit.position, refl_dir, ...)

    # Step 5: Refraction
    if ior != 1.0:
        refracted_dir = refract(ray_direction, normal, 1.0 / ior)
        # Step through volume, re-bending at each voxel boundary
        color += trace_heterogeneous_volume(hit.position, refracted_dir, octree, ior)

    return color

def trace_heterogeneous_volume(origin, direction, octree, current_ior):
    # Simplified step-through logic
    # In reality, this involves stepping voxel-by-voxor and checking IOR changes
    # Snell's law: n1 * sin(theta1) = n2 * sin(theta2)
    # Re-bend at each boundary where IOR changes
    pass
```

## Subtopics

### Implicit Octree Storage
Stores the octree structure implicitly using Morton codes, avoiding explicit pointers. [Implicit Octree Storage](939-implicit-octree-storage.md)

### Implicit Octree Traversal
Optimizes the traversal of implicit octrees for ray casting. [Implicit Octree Traversal](1220-implicit-octree-traversal.md)

### OpenVDB (VDB)
A sparse data structure for volumetric data, often used in VFX. [OpenVDB (VDB)](1508-openvdb-vdb.md)

### Traversal Acceleration
General techniques for speeding up ray traversal in sparse structures. [Traversal Acceleration](10779-traversal-acceleration.md)

## Operating characteristics
*   **Performance:** On an NVIDIA GTX 670 at 1024³ resolution:
    *   768x768: 85 fps (no effects), 65 fps (shadows), 59 fps (reflection), 32 fps (translucency) [S1, unofficial].
    *   1024x1024: 54 fps (no effects), 42 fps (shadows), 38 fps (reflection), 20 fps (translucency) [S1, unofficial].
    *   1920x1080: 26 fps (no effects), 19 fps (shadows), 17 fps (reflection), 8 fps (translucency) [S1, unofficial].
*   **Scaling:** Frame rate halves every time pixel count doubles past 1024x1024. Translucency performance drops to single digits at 1080p. Octree resolution scaling is efficient: each step up in resolution (8x voxels) costs only ~15% frame rate [S1, unofficial].
*   **Memory Usage:** 34 MB at 512³, 137 MB at 1024³, 544 MB at 2048³ [S1, unofficial].
*   **Limitations:** One thread per pixel causes warp divergence, making translucency expensive. Voxel cubes cause self-shadowing artifacts on smooth surfaces, requiring normal-aware ray casting which increases cost [S1, unofficial].

## Background
This method was developed as a BSc dissertation in Computer Games Programming at the University of Derby (2013-2014) by Catchouli [S1, unofficial]. It addresses the limitation of rasterization, which only handles surfaces, by enabling real-time GPU ray tracing for volumetric effects like clouds, smoke, and glass with internal structures. It builds on the efficient sparse voxel octree traversal algorithm by Laine and Karras (2010) [S1, unofficial].

## Sources
[S1] catchouli/VolumeRender, https://github.com/catchouli/VolumeRender, fetched 2026-10-26 (unofficial)
