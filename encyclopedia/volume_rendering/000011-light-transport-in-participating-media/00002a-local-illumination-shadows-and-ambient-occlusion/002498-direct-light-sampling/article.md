# Direct Light Sampling

Also called: Direct Illumination Sampling, Iluminación directa

## Overview
Direct Light Sampling is a technique for computing direct illumination in participating media and scenes by explicitly shooting a ray from an intersection point toward a light source to determine visibility and light contribution [S1, S2, unofficial]. It solves the problem of accurately modeling first-order scattering (in-scattering) and shadows by testing occlusion along the path to the light, distinguishing itself from methods that rely on random walk termination or global opacity interpolation [S1, S2, unofficial]. Unlike [Light Ray Sampling](4885-light-ray-sampling.md), which integrates density along a fixed direction, Direct Light Sampling explicitly selects light sources and verifies geometric visibility, often using a power heuristic for unbiased estimation [S2, unofficial].

## How it works
**Inputs:**
- Ray $r$ with origin and direction.
- Set of scene objects $O$.
- Light source $l$ (or a list of lights).
- Bounce counter $b$.
- Current accumulated color $c$ (initially 1 or 0 depending on implementation).

**Outputs:**
- Final color contribution from the light path.

**Data Structures:**
- `LightHit`: A structure containing:
  - `colour`: Radiance/Color of the light ($Vec3f$).
  - `pdf`: Probability density function of the sampling ($float$).
  - `direction`: Unit vector from surface to light ($Vec3f$).
  - `distance`: Distance to light source ($float$).
- `ScatterEvent`: Interaction point data including position, normal, and output direction.

**Steps:**
1. **Initialize:** Set hit flag $hl = false$ and accumulated color $c = 1$ (or 0 if accumulating from background) [S1, unofficial].
2. **Ray Intersection:** Find the closest object $o \in O$ intersected by ray $r$ at distance $t > 0$. If no intersection, return background color modulated by $c$ [S1, unofficial].
3. **Light Direction:** Calculate direction to light: $\mathbf{dl} = \text{normalize}(\text{position}(l) - \mathbf{p})$, where $\mathbf{p}$ is the intersection point [S1, S2, unofficial].
4. **Shadow Ray:** Create shadow ray $r_l$ from $\mathbf{p}$ in direction $\mathbf{dl}$ [S1, S2, unofficial].
5. **Light Intersection:** Test intersection between $r_l$ and light $l$ to get distance $t_l$ [S1, S2, unofficial].
6. **Occlusion Test:** Find minimum distance $t_o$ to any object $o \in O$ along $r_l$ (excluding the light itself if it is also an object, or treating lights as transparent for occlusion). Let $t_{min} = \min \{ t \mid o \in O \land t = \text{intersection test}(r_l, o) \land t > 0 \}$ [S1, S2, unofficial].
7. **Visibility Check:** Update hit flag: $hl \leftarrow hl \lor (t_l \le t_{min})$. If the light is hit and closer than any occluding object, the point is directly illuminated [S1, S2, unofficial].
8. **Illumination Calculation:**
   - If using a sampler to select lights (for multiple lights), select a random light $l_i$ not equal to the current hit light [S2, unofficial].
   - Calculate light contribution: $\text{weight} = \text{powerHeuristic}(1, \text{lightHit.pdf}, 1, \text{pdf})$ [S2, unofficial].
   - Add to direct light: $\text{directLight} += \text{material} \rightarrow \text{evaluate}(\text{interaction}) \times \text{lightHit.colour} \times \text{weight} / \text{lightHit.pdf}$ [S2, unofficial].
   - Scale by total number of lights if uniform sampling is used [S2, unofficial].
9. **Recursive Bounce:** If bounce counter $b > 0$, compute valid bounce direction $\mathbf{d}$, create new ray $r' = \text{ray}(\mathbf{p}, \mathbf{d})$, update $c \leftarrow c \circ \text{color material}(\mathbf{p}, o)$, and recurse [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_light_sampling(r, objects, lights, bounce_count, accumulated_color):
    """
    Computes direct light contribution by shooting a shadow ray to a light source.
    
    Args:
        r: Ray object with origin and direction.
        objects: List of scene objects.
        lights: List of light sources.
        bounce_count: Integer, remaining bounces.
        accumulated_color: Vector3, accumulated color so far.
        
    Returns:
        Vector3: Final color contribution.
    """
    if bounce_count < 0:
        # If no bounces left, return background modulated by accumulated color
        # If direct light was hit (hl), add light color
        # Note: hl state must be tracked or passed. 
        # Simplified: assume this function is called after visibility check.
        return accumulated_color * get_background_color(r.direction)

    # Find closest intersection
    closest_obj = None
    closest_t = infinity
    for obj in objects:
        t = obj.intersection_test(r)
        if t > 0 and t < closest_t:
            closest_t = t
            closest_obj = obj

    if closest_obj is None:
        return accumulated_color * get_background_color(r.direction)

    # Intersection point
    p = r.origin + closest_t * r.direction
    
    # Select a light (simplified: assume single light or uniform sampling)
    # In a full path tracer, a light is selected via sampler
    light = select_light(lights) # Placeholder for sampler logic
    
    # Calculate direction to light
    dl = normalize(light.position - p)
    
    # Create shadow ray
    shadow_ray = Ray(p, dl)
    
    # Test intersection with light
    t_light = light.intersection_test(shadow_ray)
    
    # Test occlusion by other objects
    t_occlusion = infinity
    for obj in objects:
        t = obj.intersection_test(shadow_ray)
        if t > 0 and t < t_occlusion:
            t_occlusion = t
            
    # Check if light is visible
    is_visible = (t_light < t_occlusion)
    
    if is_visible:
        # Calculate light contribution
        # For PointLight, pdf = 4 * pi * d^2
        # For DirectionalLight, pdf = 1 (or handled differently)
        light_hit = light.illuminate(ScatterEvent(p, closest_obj.normal, r.direction))
        
        # Power heuristic for unbiased estimation
        # Assuming 1 sample from light and 1 sample from path
        weight = power_heuristic(1, light_hit.pdf, 1, 1.0) # pdf of path direction approx 1 for simplicity
        
        # Material evaluation
        material_color = closest_obj.material.evaluate(p, closest_obj.normal, r.direction)
        
        # Add contribution
        contribution = material_color * light_hit.colour * weight / light_hit.pdf
        
        # Scale by number of lights if uniform sampling
        contribution *= len(lights)
        
        accumulated_color += contribution

    # Recursive bounce
    if bounce_count > 0:
        d = get_valid_bounce_direction(p, closest_obj.normal, r.direction)
        new_ray = Ray(p, d)
        new_accumulated = accumulated_color * closest_obj.material.color_at(p)
        return direct_light_sampling(new_ray, objects, lights, bounce_count - 1, new_accumulated)
        
    return accumulated_color

def power_heuristic(n, pdf_a, m, pdf_b):
    """
    Calculates the power heuristic weight.
    """
    denom = n * pdf_a + m * pdf_b
    if denom == 0:
        return 0
    return (n * pdf_a) / (denom * pdf_a)
```

**Parameters:**
- `bounce_count`: Typically 1-5 for direct illumination only, higher for full path tracing [S1, unofficial].
- `sigma` (Gaussian filter): Used post-processing to reduce noise, e.g., $\sigma = 4$ [S1, unofficial].

**Edge Cases:**
- **No Lights:** Return zero or background [S2, unofficial].
- **Self-Intersection:** Shadow ray origin should be offset slightly along normal to avoid self-intersection [G].
- **Precision:** Floating point errors in $t_l \le t_{min}$ comparison can cause flickering; use a small epsilon [G].

## Subtopics

### [Light Sampling for NEE](2616-light-sampling-for-nee.md)
Reduces variance in direct illumination by explicitly sampling light sources, decoupling light selection from path vertex generation.

### [Direct Illumination Sampling with Mesh Intersection](4450-direct-illumination-sampling-with-mesh-intersection.md)
Handles direct illumination for polygonal scenes using explicit triangle meshes, distinguishing itself from SDF-based approaches.

### [Light Ray Sampling](4885-light-ray-sampling.md)
Computes direct illumination by stepping along the direction to a distant light, integrating density for soft volumetric shadows.

### [Direct Light Contribution Estimation](5411-direct-light-contribution-estimation.md)
Separates attenuation into high-resolution local and low-resolution global segments for accurate first-order scattering.

## Operating characteristics
- **Time Complexity:** $O(N \cdot L)$ per pixel, where $N$ is the number of bounces and $L$ is the number of lights (if testing all) or $O(N)$ if using a sampler to select one light [S1, S2, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space per ray, excluding scene data structures [S1, unofficial].
- **Performance:** Limited by hardware constraints; launching infinite rays uniformly is not feasible for real-time FPS (>15 FPS) [S1, unofficial]. Post-processing filters (Gaussian, bilinear) are often applied to reduce noise [S1, unofficial].
- **Strengths:** Accurate shadowing and direct illumination; simple to implement [S1, unofficial].
- **Limitations:** High variance (noise) in complex scenes; requires many samples for convergence [S1, unofficial].

## Background
Originated in early ray tracing literature for visualizing shadows and light contributions in participating media and scenes [S1, unofficial]. The method is described in the context of Sparse Voxel Octree compression and efficiency studies, where direct illumination is added to improve visual quality by shooting rays from intersection points to light sources [S1, unofficial]. It is also implemented in educational path tracers for handling different light types and ensuring visibility [S2, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] thewildnath/PathTracer, https://github.com/thewildnath/PathTracer, fetched 2026-10-26 (unofficial)
