# Performance Optimizations for SDF Ray Marching

Also called: Optimisations, SDF Ray Marching Optimizations

## Overview
Performance Optimizations for SDF Ray Marching refers to a collection of techniques designed to reduce the computational cost of rendering Signed Distance Field (SDF) scenes in real-time. It solves the problem of high sample counts and expensive SDF evaluations required for high-resolution or complex geometry. It is distinguished from sibling methods like [Beam Optimization](932-beam-optimization.md) and [Block Culling](5431-block-culling.md) by focusing on per-ray and per-pixel efficiency gains (such as step size adaptation, resolution scaling, and early-outs) rather than global spatial culling or beam-based traversal. It builds on [Acceleration and Optimization](6222-acceleration-and-optimization.md) by applying specific heuristics to the sphere tracing loop.

## How it works
The method consists of several independent optimizations that can be applied to the standard sphere tracing algorithm.

### Inputs and Outputs
- **Inputs:** Scene SDF function $S(p)$, camera parameters, resolution, and optimization flags.
- **Outputs:** Rendered image with preserved visual fidelity at higher frame rates.

### Data Structures
No new complex data structures are required for the core optimizations described below. They rely on standard variables:
- `ray_origin`: `vec3`
- `ray_direction`: `vec3` (normalized)
- `t`: `float` (current ray distance)
- `d`: `float` (SDF value at current position)
- `MAX_STEPS`: `int`
- `EPSILON`: `float`

### Steps

#### 1. Early-Out Bounding Volumes
Surround expensive sub-scenes or complex geometry with a fast, cheap bounding volume SDF (typically a sphere). Before evaluating the expensive SDF, check the bounding volume. If the distance to the bounding volume is greater than the current ray step size, skip the expensive evaluation for that step.

**Formula:**
Let $B(p)$ be the bounding sphere SDF centered at $C$ with radius $R$.
$$ B(p) = \|p - C\| - R $$
If $B(ray\_origin + t \cdot ray\_direction) > d_{step}$, skip the detailed SDF evaluation for this step.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_with_bounding_volume(ray_origin, ray_direction, max_steps, epsilon):
    t = 0.0
    for i in range(max_steps):
        pos = ray_origin + t * ray_direction
        
        # Check bounding volume first
        dist_to_center = length(pos - bounding_center)
        dist_bounding = dist_to_center - bounding_radius
        
        if dist_bounding > d_step: # d_step is the previous SDF value or a heuristic step
            # Skip expensive SDF evaluation, assume empty or use a coarse approximation
            # Note: In strict sphere tracing, we must ensure we don't overshoot.
            # A safe early-out usually implies the bounding volume itself defines the step.
            # If the bounding volume is a sphere, the step is dist_bounding.
            t += dist_bounding
            continue
            
        # Evaluate expensive SDF
        d = evaluate_expensive_sdf(pos)
        
        if d < epsilon:
            return hit(pos, d)
            
        if t > max_distance:
            return miss()
            
        t += d
    return miss()
```
*[G] Note: Standard sphere tracing requires $t += d$. If using a bounding volume early-out, the step size must be conservative. If the bounding volume is a sphere, $d_{step} = B(pos)$ is a valid lower bound for the distance to the inner geometry, allowing safe advancement.* [S1, unofficial]

#### 2. Level of Detail (LOD) for Distance
Adjust `MAX_STEPS` and `EPSILON` based on the distance of the geometry from the camera. For distant geometry, the visual difference between a coarse and fine trace is imperceptible.

**Parameters:**
- Reduce `MAX_STEPS` for distant objects.
- Increase `EPSILON` (e.g., from $10^{-4}$ to $10^{-2}$) for distant objects.
- Threshold: Differences are imperceptible beyond 10 scene units. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_lod_params(distance):
    if distance > 10.0:
        return max_steps=32, epsilon=0.01
    else:
        return max_steps=64, epsilon=0.0001
```
[S1, unofficial]

#### 3. Resolution Scaling
Render the scene at half the target resolution (e.g., 50% width and height) and upscale the result using a fast bilinear or bicubic filter. This is described as the "most effective single optimisation for real-time SDF rendering at larger resolutions." [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# In the fragment shader or post-process
if use_half_res:
    uv = gl_FragCoord.xy / vec2(screen_width * 0.5, screen_height * 0.5)
    # Perform ray march at lower res
    color = ray_march(uv)
    # Upscale in post-process or via texture sampling
    final_color = texture(upscaler_texture, uv)
else:
    uv = gl_FragCoord.xy / vec2(screen_width, screen_height)
    final_color = ray_march(uv)
```
[S1, unofficial]

#### 4. Over-Relaxation
Multiply the step size $d$ by a constant factor $k$ where $1.1 \le k \le 1.6$. This allows the ray to "jump" further, reducing the number of iterations.

**Risk:** This risks artifacts near concave edges or thin geometry, as the ray may overshoot the surface. It is best suited for open scenes or convex objects. [S1, unofficial]

**Formula:**
$$ t_{new} = t_{old} + d \cdot k $$
where $k \in [1.1, 1.6]$. [S1, unofficial]

### Edge Cases and Pitfalls
- **Over-Relaxation Artifacts:** Using $k > 1.0$ can cause the ray to skip over thin geometry or miss concave details, leading to visual holes or incorrect normals. [S1, unofficial]
- **Bounding Volume Precision:** If the bounding volume is not tight, the early-out may skip valid geometry. If it is too tight, the overhead of checking it may negate the benefit.
- **LOD Pop-in:** Abrupt changes in `MAX_STEPS` or `EPSILON` can cause visible popping. Smooth interpolation of these parameters with distance is recommended. [G]

## Subtopics

### Pre-marched Shadow and Ambient Occlusion Baking
A technique to improve visual fidelity and performance by pre-calculating shadow and ambient occlusion data into the volume texture, leveraging the static nature of the cloud volume [S3, unofficial]. It differs from standard runtime ray marching by baking lighting data into a 3D texture, allowing runtime shadow lookup with a single `tex3D` call [S3, unofficial].
- **Baking Shadow:** Pre-march a shadow ray through the volume. Due to low volume resolution (e.g., 4 shadow samples per cloud sample), this can be done in real-time, allowing dynamic time-of-day changes [S3, unofficial].
- **Baking Ambient Occlusion:** Pre-march ambient occlusion into the volume texture to add shading without runtime cost [S3, unofficial].
- **Runtime Lookup:** Use the baked shadow texture to get volumetric cloud shadows with a single `tex3D` lookup, matching the cloud geometry perfectly [S3, unofficial].

### MMX Tri-linear Interpolation
Using MMX instruction set extensions to exploit instruction-level parallelism for tri-linear interpolation, executing multiple low-resolution fixed-point operations in parallel [S4]. It differs from standard floating-point interpolation by using integer arithmetic and specific MMX instructions (PMULHW, PMULLW, PMADDWD, PSUBW) to perform tri-linear interpolation in only 37 instructions [S4].
- **Performance:** On Pentium processors, MMX provides 60-80% faster performance. On Pentium-II, it provides 0% improvement and may worsen performance due to pipeline stalls [S4].
- **Pitfall:** Performance on Pentium-II does not improve and may worsen compared to the non-MMX version [S4].

### Symmetry Optimization for SDF Evaluation
Evaluate only a subset of primitives based on object symmetry to reduce computational cost [S5, unofficial]. For example, evaluate only 4 gear pieces instead of 18, and within each gear, evaluate only a single tooth/dent instead of 12 [S5, unofficial]. It is used in the 'Sphere Gears' Shadertoy example (2019) as an optimization exercise [S5, unofficial].

### Ray Differentials for Texture Filtering
A technique using ray differentials to do proper texture filtering in a raymarching context [S5, unofficial]. It ensures that procedural textures sampled during ray marching are filtered correctly to avoid aliasing, similar to how rasterization handles texture filtering. It is used in the 'Arlo' Shadertoy example (2015) for procedural modeling [S5, unofficial].

## Operating characteristics
- **Time Complexity:** The optimizations reduce the average number of SDF evaluations per pixel. Early-outs and LOD reduce the constant factor in the loop. Resolution scaling reduces the total pixel count by 4x (for half resolution).
- **Memory Complexity:** Pre-marched baking (S3, unofficial) increases memory usage by storing shadow and AO textures. MMX optimization (S4) has no additional memory cost.
- **Measured Performance:**
    - Replacing floating-point with integer arithmetic yields 4 to 6 times faster performance on older hardware (Pentium-II) [S4].
    - Hand-coded address generation for tri-linear interpolation improves performance by up to 15% [S4].
    - Rendering at half resolution is described as the "most effective single optimisation" for real-time SDF rendering at larger resolutions [S1, unofficial].
- **Strengths:** Significant frame rate improvements, especially on older hardware or for complex procedural SDFs.
- **Limitations:** Over-relaxation introduces artifacts. Baking requires static scenes. MMX is obsolete on modern GPUs.
- **When to use:** Real-time rendering of complex SDF scenes, VR applications, or hardware with limited ALU throughput.
- **When not to use:** When visual precision is critical (avoid over-relaxation), or when the scene is dynamic and baking is not feasible.

## Background
The techniques described originate from various sources in the ray marching community. Early-out bounding volumes and LOD strategies are discussed in [S1, unofficial] and [S2, unofficial]. The use of MMX for volume rendering was detailed in "I/O-Conscious Volume Rendering" [S4]. Symmetry and ray differentials were popularized by Inigo Quilez in his articles and Shadertoy examples [S5, unofficial]. Pre-marched baking for volumetric clouds was described in [S3, unofficial].

## Sources
[S1] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S2] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S3] Rendering volumetric clouds using signed distance fields, https://blog.uhawkvr.com/rendering/rendering-volumetric-clouds-using-signed-distance-fields/, fetched 2026-10-26 (unofficial)
[S4] I/O-Conscious Volume Rendering, https://doi.org/10.2312/vissym/vissym01/263-272, fetched 2026-10-26
[S5] Raymarching Distance Fields, https://iquilezles.org/articles/raymarchingdf/, fetched 2026-10-27 (unofficial)
