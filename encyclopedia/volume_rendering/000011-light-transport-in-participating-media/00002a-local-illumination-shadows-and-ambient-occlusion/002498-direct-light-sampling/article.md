# Direct Light Sampling

Also called: Direct Illumination Sampling, Iluminación directa

## Overview
Direct Light Sampling is a local illumination technique that explicitly shoots a ray from a surface or volume intersection point toward a light source to determine direct visibility and contribution [S1, S2, unofficial]. It solves the problem of computing first-order lighting (shadows and direct illumination) by testing occlusion between the shading point and the light, distinguishing itself from global illumination methods by ignoring indirect bounces [S1, unofficial]. Unlike [Shadow Ray Casting](2796-shadow-ray-casting.md), which may use fixed directions or simplified visibility tests, this method explicitly samples light sources (or uses a specific light) and calculates the geometric probability density function (PDF) for unbiased estimation when combined with multiple lights [S2, unofficial].

## How it works
**Inputs:**
- Intersection point $\mathbf{p}$ and its normal $\mathbf{n}$ (or volume position).
- List of scene objects $\mathcal{O}$ for occlusion testing.
- List of light sources $\mathcal{L}$.
- Current light being traced or a sampler for selecting a light [S2, unofficial].

**Outputs:**
- Direct illumination color contribution $L_{direct}$ at point $\mathbf{p}$.

**Data Structures:**
- `LightHit`: A structure containing:
  - `colour`: Radiance/Color of the light ($\mathbf{C}_L$).
  - `pdf`: Probability density of the sampling direction ($p(\mathbf{l})$).
  - `direction`: Unit vector from $\mathbf{p}$ to the light ($\mathbf{l}$).
  - `distance`: Distance from $\mathbf{p}$ to the light source ($d$).
- `Ray`: Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- `Intersection`: Contains distance $t$ and object reference.

**Steps:**

1.  **Light Selection:**
    - If the implementation uses a single fixed light (e.g., for debugging or simple scenes), select that light $\mathbf{l}$ [S1, unofficial].
    - If using multiple lights, select a random light from $\mathcal{L}$ using a sampler, ensuring it is not the same as the light that generated the current path segment (if applicable) [S2, unofficial].

2.  **Direction Calculation:**
    - Calculate the direction vector to the light source:
      $$ \mathbf{l}_{dir} = \text{normalize}(\text{position}(\mathbf{l}) - \mathbf{p}) $$
      [S1, unofficial]
    - For point lights, the PDF is derived from the solid angle subtended by the light. For a point light at distance $d$, the PDF is:
      $$ p(\mathbf{l}) = \frac{1}{4\pi d^2} $$
      [S2, unofficial]

3.  **Shadow Ray Creation:**
    - Create a shadow ray $\mathbf{r}_s$ originating from $\mathbf{p}$ in direction $\mathbf{l}_{dir}$:
      $$ \mathbf{r}_s = \text{Ray}(\mathbf{p}, \mathbf{l}_{dir}) $$
      [S1, unofficial]

4.  **Occlusion Test:**
    - Test for intersection between $\mathbf{r}_s$ and the light source $\mathbf{l}$ to get distance $t_l$ [S1, unofficial].
    - Test for intersection between $\mathbf{r}_s$ and all scene objects $\mathcal{O}$ to find the minimum distance $t_{min}$ among all occluders:
      $$ t_{min} = \min \{ t \mid \mathbf{o} \in \mathcal{O} \land t = \text{intersection test}(\mathbf{r}_s, \mathbf{o}) \land t > 0 \} $$
      [S1, unofficial]
    - Determine visibility flag $h_l$:
      $$ h_l = (t_l \le t_{min}) $$
      [S1, unofficial]

5.  **Contribution Calculation:**
    - If the light is visible ($h_l$ is true), calculate the direct light contribution.
    - Retrieve the light's color/intensity and the material's reflectance (BRDF/BTDF) $f_r(\mathbf{p}, \mathbf{l}_{dir}, \mathbf{v})$ [S2, unofficial].
    - Apply the power heuristic if combining with other sampling strategies (e.g., multiple importance sampling), though for pure direct sampling with one light per pixel, the weight is often 1 or scaled by $1/N_{lights}$ [S2, unofficial].
    - The direct light contribution is:
      $$ L_{direct} = f_r(\mathbf{p}, \mathbf{l}_{dir}, \mathbf{v}) \cdot \mathbf{C}_L \cdot \frac{1}{p(\mathbf{l})} \cdot h_l $$
      [S2, unofficial]
    - If using uniform random light selection among $N$ lights, scale the result by $N$ [S2, unofficial].

6.  **Edge Cases:**
    - If no lights exist or the selected light is the source of the current ray (self-interaction), return zero contribution [S2, unofficial].
    - If the material is perfectly specular/mirror-like and the reflection direction does not align with the light, contribution may be zero [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_light_sampling(p, n, objects, lights, sampler):
    """
    Computes direct illumination at point p.
    p: intersection point (Vec3)
    n: surface normal (Vec3)
    objects: list of scene objects for occlusion testing
    lights: list of light sources
    sampler: random number generator/sampler
    """
    # 1. Light Selection
    # Select a random light from the scene's lights
    # Ensure it is not the same as the hitLight if tracing a path
    light = sampler.sample_light(lights)
    
    if light is None:
        return Vec3(0, 0, 0)

    # 2. Direction Calculation
    light_pos = light.position()
    l_dir = normalize(light_pos - p)
    distance = length(light_pos - p)
    
    # Calculate PDF for point light
    # PDF = 1 / (4 * pi * distance^2)
    pdf = 1.0 / (4.0 * PI * distance * distance)
    
    # 3. Shadow Ray Creation
    shadow_ray = Ray(p, l_dir)
    
    # 4. Occlusion Test
    # Test intersection with the light source
    t_light = shadow_ray.intersect(light)
    
    # Test intersection with all scene objects
    min_t_occluder = infinity
    for obj in objects:
        t_obj = shadow_ray.intersect(obj)
        if t_obj > 0 and t_obj < min_t_occluder:
            min_t_occluder = t_obj
            
    # Visibility check: is the light closer than any occluder?
    is_visible = (t_light <= min_t_occluder)
    
    if not is_visible:
        return Vec3(0, 0, 0)
        
    # 5. Contribution Calculation
    # Get light color and intensity
    light_color = light.color() * light.intensity()
    
    # Evaluate material BRDF
    # f_r is the Bidirectional Reflectance Distribution Function
    # For a simple diffuse material: f_r = albedo / pi
    albedo = material.albedo(p)
    brdf = albedo / PI
    
    # Calculate direct light contribution
    # L_direct = f_r * C_L * (1 / pdf)
    # Note: In a multiple light scenario, this result should be scaled by N_lights
    # if uniform sampling is used.
    direct_contribution = brdf * light_color * (1.0 / pdf)
    
    return direct_contribution
```

## Subtopics

### [Light Sampling for NEE](2616-light-sampling-for-nee.md)
Extends direct sampling to Nearest Energy Estimation for unbiased path tracing in participating media, handling PDFs for connection points.

### [Direct Illumination Sampling with Mesh Intersection](4450-direct-illumination-sampling-with-mesh-intersection.md)
Specialized for polygonal meshes, using explicit triangle intersections rather than implicit volume or SDF tests.

### [Light Ray Sampling](4885-light-ray-sampling.md)
Steps along the light direction through a volume to integrate density, creating soft volumetric shadows rather than binary visibility.

### [Direct Light Contribution Estimation](5411-direct-light-contribution-estimation.md)
Separates attenuation into local high-resolution and global low-resolution segments for efficient direct light in volumes.

## Operating characteristics
- **Time Complexity:** $O(N_{lights} \cdot N_{objects})$ per pixel if testing all lights and objects, or $O(N_{objects})$ per light if one light is selected per sample. In practice, often $O(1)$ if only one dominant light is sampled per pixel.
- **Memory Complexity:** $O(1)$ auxiliary memory, excluding the scene graph storage.
- **Strengths:** Simple to implement; provides sharp, correct shadows for point and directional lights; efficient for scenes with few dominant lights.
- **Limitations:** Noisy for complex lighting environments with many small lights (requires many samples); does not account for indirect illumination (bounces); binary shadow edges can appear aliased without supersampling or filtering [S1, unofficial].
- **When to use:** Real-time rendering with simple lighting; offline rendering where direct lighting is the primary concern; scenes with few light sources.
- **When not to use:** Scenes requiring soft shadows from large area lights (requires multiple samples or specific area light sampling); scenes dominated by indirect lighting (requires path tracing or photon mapping).

## Background
The method originates from basic ray tracing algorithms developed in the late 1970s and 1080s, where direct illumination was computed by shooting shadow rays to light sources [S1, unofficial]. The specific implementation details cited here are drawn from educational resources on Sparse Voxel Octrees and path tracing implementations [S1, S2, unofficial]. The term "Iluminación directa" is used in Spanish-language technical reports describing these optimizations [S1, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S2] thewildnath/PathTracer, https://github.com/thewildnath/PathTracer, fetched 2026-10-26 (unofficial)
