# Butterfly Ray Marching with SDFs

Also called: Butterfly SDF Ray Marching, Swarm SDF Rendering

## Overview
Butterfly Ray Marching with SDFs renders a swarm of butterflies by combining Signed Distance Functions (SDFs) for individual anatomical parts (body, wings, antennae) with ray marching in a transformed local coordinate space to handle dynamic wing flapping and orientation [S1, unofficial]. It sets itself apart from standard SDF ray marching by using a non-rigid "un-bend" transform to animate wings and a specific stability clamping strategy for step sizes, building on the soft accumulation techniques of [Ray Marching with SDFs and Soft Accumulation](5001-ray-marching-with-sdfs-and-soft-accumulation.md).

## How it works
**Inputs:**
- Ray origin $o$ and direction $d$ in world space.
- Time $t$ and phase offset.
- Swarm data array `SWARM` containing position and velocity vectors for each butterfly.
- Parameters: `MARCH_STEPS` (max iterations), `SWARM_SIZE` (number of butterflies).

**Outputs:**
- Final pixel color $C_{out}$.

**Data Structures:**
- `SWARM[i]`: A 4-component vector (or similar structure) storing the base position and velocity/orientation data for butterfly $i$.
- `mapButterfly(p, t, phase)`: Returns a 2-component vector `h` where `h.x` is the signed distance and `h.y` is the material ID (0 for wings, 1 for body).

**Step-by-Step Method:**

1.  **Local Space Transformation:**
    For each butterfly $i$ in the swarm:
    - Retrieve velocity vector $v$ and base position from `SWARM[i]`.
    - Determine the forward vector. If $|v_{xy}| < 10^{-4}$, derive forward from the cross product of the dorsal axis and $(1, 0, 0)$ to avoid singularity [S1, unofficial].
    - Construct a basis matrix from the velocity vector and a dorsal axis (interpolated between world up and camera direction).
    - Apply a roll rotation to the basis.
    - Transform the world ray origin $o$ and direction $d$ into the butterfly's local space:
      $$ o_{local} = B^{-1} (o - p_{butterfly}) / s $$
      $$ d_{local} = B^{-1} d $$
      where $B$ is the basis matrix and $s$ is the scale factor [S1, unofficial].

2.  **Sphere Intersection:**
    - Compute the intersection of the transformed ray with a unit sphere bounding the butterfly:
      $$ b = \text{dot}(o_{local}, d_{local}) $$
      $$ disc = b^2 - \text{dot}(o_{local}, o_{local}) + 1.45^2 $$
    - If $disc < 0$, skip this butterfly.
    - Calculate $t_{max} = -b + \sqrt{disc}$. If $t_{max} \le 0$, skip.

3.  **Ray Marching Loop:**
    - Initialize $travel = \max(-b - \sqrt{disc}, 0) + \text{dither}$.
    - Iterate $k$ from 0 to `MARCH_STEPS`:
      - If $travel > t_{max}$, break.
      - Compute point $p = o_{local} + d_{local} \cdot travel$.
      - Evaluate $h = \text{mapButterfly}(p, t, \text{phase})$.
      - If $h.x < 0.0022$, a hit is found. Set $hit = travel$, $hp = p$, $mat = h.y$, and break.
      - Advance step:
        $$ travel \leftarrow travel + \text{clamp}(h.x \cdot 0.62, 0.0035, 0.22) $$
      - Note: The step size is clamped because the un-bend transform is not rigid [S1, unofficial].

4.  **SDF Evaluation (`mapButterfly`):**
    - **Body SDF:**
      - Clamp $z$ to $[-0.38, 0.30]$.
      - Compute normalized height $u$.
      - Calculate radius $r$ using a base term, a Gaussian bump for the thorax, and an offset.
      - Return distance to cylinder surface defined by $x, y \cdot 1.12, z - z_{offset}$.
    - **Antenna SDF:**
      - Take absolute $x$.
      - Clamp normalized $z$ to $[0, 1]$.
      - Compute cubic Bezier-like control point $c$.
      - Return distance to line segment with radius interpolating between 0.0045 and 0.014 based on $u$.
    - **Wing SDF:**
      - Compute wing distance.
    - **Combine:**
      - Body SDF = $\min(\text{Body}, \text{Antenna})$.
      - If $\text{Wing} < \text{Body}$, return $(\text{Wing}, 0)$.
      - Else, return $(\text{Body}, 1)$.

5.  **Normal Calculation:**
    - Compute normal $n$ using central differences on `mapButterfly` with $\epsilon = 0.0018$ [S1, unofficial].

6.  **Shading:**
    - Transform hit point $hp$ back to world space if necessary for lighting.
    - **Wings (Material 0):**
      - Transform to wing space.
      - Calculate outline and base color.
      - Lambertian lighting: $L = 0.22 + 0.78 \max(\text{dot}(n, \text{sun}), 0)$.
      - Subsurface scattering approximation: $S = \text{base} \cdot \max(-\text{dot}(n, \text{sun}), 0) \cdot \max(\text{dot}(rd, \text{sun}), 0)^{2.2} \cdot 6.0$.
      - Rim lighting: $R = \text{mix}(\text{SUNCOL}, \text{AMBER\_BRIGHT}, 0.55) \cdot \text{pow}(1.0 - \text{abs}(\text{dot}(n, -d)), 3.0) \cdot 0.24$.
      - Specular: $Spec = \text{SUNCOL} \cdot \text{pow}(\max(\text{dot}(\text{reflect}(-\text{sun}, n), -d), 0), 28.0) \cdot 0.14$.
      - Final: $shade = \text{base} \cdot (L + 0.18) + S + \text{base} \cdot \text{through} \cdot 0.60 + R + Spec$.
    - **Body (Material 1):**
      - Lambertian lighting: $L = 0.25 + 0.75 \max(\text{dot}(n, \text{sun}), 0)$.
      - Fuzz factor: $F = 0.5 + 0.5 \sin(hp.z \cdot 180.0) \sin(hp.x \cdot 150.0)$.
      - Final: $shade = \text{vec3}(0.070, 0.042, 0.032) \cdot L \cdot (0.8 + 0.3 F) + \text{SUNCOL} \cdot \text{pow}(1.0 - \text{abs}(\text{dot}(n, -d)), 2.5) \cdot 0.55$.

7.  **Compositing:**
    - Calculate distance $dist = |ro - origin|$.
    - Haze factor: $H = \exp(-0.055 \cdot \max(dist - 2.0, 0.0))$.
    - $col = \text{mix}(\text{background}(rd, uv), shade, H)$.
    - Apply tonemapping and gamma correction.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def butterfly_ray_march(ro, rd, t, phase, swarm_data, march_steps):
    """
    ro: Ray origin (vec3)
    rd: Ray direction (vec3)
    t: Time (float)
    phase: Phase offset (float)
    swarm_data: List of butterfly data (vec4)
    march_steps: Max iterations (int)
    """
    final_color = vec3(0.0)
    
    for i in range(len(swarm_data)):
        data = swarm_data[i]
        pos = data.xyz
        vel = data.w # Assuming velocity/orientation data is packed here or derived
        
        # 1. Local Space Transformation
        # Construct basis from velocity and dorsal axis
        forward = normalize(vel.xy) if length(vel.xy) > 1e-4 else cross(vec3(0,1,0), vec3(1,0,0))
        dorsal = mix(vec3(0,1,0), normalize(ro - pos), 0.5) # Simplified dorsal logic
        basis = construct_basis(forward, dorsal)
        
        # Transform ray to local space
        o_local = transform_point(basis, ro - pos) / scale
        d_local = transform_vector(basis, rd)
        
        # 2. Sphere Intersection
        b = dot(o_local, d_local)
        disc = b * b - dot(o_local, o_local) + 1.45 * 1.45
        
        if disc < 0.0:
            continue
            
        sq = sqrt(disc)
        t_max = -b + sq
        if t_max <= 0.0:
            continue
            
        travel = max(-b - sq, 0.0) + dither()
        hit_t = -1.0
        hit_p_local = vec3(0.0)
        hit_mat = 0.0
        
        # 3. Ray Marching
        for k in range(march_steps):
            if travel > t_max:
                break
                
            p_local = o_local + d_local * travel
            h = map_butterfly(p_local, t, phase)
            
            if h.x < 0.0022:
                hit_t = travel
                hit_p_local = p_local
                hit_mat = h.y
                break
                
            step = clamp(h.x * 0.62, 0.0035, 0.22)
            travel += step
            
        if hit_t < 0.0:
            continue
            
        # 4. Normal Calculation
        n = butterfly_normal(hit_p_local, t, phase)
        
        # 5. Shading
        shade = vec3(0.0)
        if hit_mat < 0.5:
            # Wing shading
            v = to_wing_space(vec3(abs(hit_p_local.x), hit_p_local.y, hit_p_local.z), t, phase)
            w = vec2(v.x, v.z)
            outline = wing_outline(w)
            base_col = wing_colour(w, outline, hash11(i + 5.0) * 0.35)
            
            lambert = 0.22 + 0.78 * max(dot(n, sun), 0.0)
            through = max(-dot(n, sun), 0.0)
            back = pow(max(dot(rd, sun), 0.0), 2.2)
            rim = pow(1.0 - abs(dot(n, -rd)), 3.0)
            
            shade = base_col * (lambert + 0.18)
            shade += base_col * through * back * 6.0
            shade += base_col * through * 0.60
            shade += mix(sun_col, amber_bright, 0.55) * rim * 0.24
            shade += sun_col * pow(max(dot(reflect(-sun, n), -rd), 0.0), 28.0) * 0.14
        else:
            # Body shading
            lambert = 0.25 + 0.75 * max(dot(n, sun), 0.0)
            fuzz = 0.5 + 0.5 * sin(hit_p_local.z * 180.0) * sin(hit_p_local.x * 150.0)
            shade = vec3(0.070, 0.042, 0.032) * lambert * (0.8 + 0.3 * fuzz)
            shade += sun_col * pow(1.0 - abs(dot(n, -rd)), 2.5) * 0.55
            
        # 6. Compositing
        dist = length(ro - pos)
        haze = exp(-0.055 * max(dist - 2.0, 0.0))
        bg_col = background(rd, uv)
        final_color = mix(bg_col, shade, haze)
        
    return final_color

def map_butterfly(p, t, phase):
    """Returns (distance, material_id)"""
    # Body SDF
    z_clamped = clamp(p.z, -0.38, 0.30)
    u = normalized_height(z_clamped)
    r = base_radius + gaussian_thorax(u) + offset
    body_dist = distance_to_cylinder(p.x, p.y * 1.12, z_clamped - z_offset, r)
    
    # Antenna SDF
    abs_x = abs(p.x)
    z_norm = clamp(normalized_z(p.z), 0.0, 1.0)
    c = cubic_bezier_control(z_norm)
    ant_dist = distance_to_segment(abs_x, z_norm, c, radius_interpolate(u))
    
    body_sdf = min(body_dist, ant_dist)
    
    # Wing SDF
    wing_dist = compute_wing_distance(p, t, phase)
    
    if wing_dist < body_sdf:
        return vec2(wing_dist, 0.0)
    else:
        return vec2(body_sdf, 1.0)
```

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$, where $N$ is `SWARM_SIZE` and $M$ is `MARCH_STEPS`. Each butterfly requires a separate ray march.
- **Memory Complexity:** $O(N)$ to store swarm data.
- **Performance:** Performance is limited by the number of butterflies and the cost of the SDF evaluation and shading per step. The clamping of step sizes ensures stability but may increase the number of steps required for convergence in non-rigid regions [S1, unofficial].
- **Strengths:** Realistic rendering of complex, animated organic shapes (wings) using SDFs. Handles transparency and subsurface scattering effects effectively.
- **Limitations:** High computational cost for large swarms. The non-rigid transform can lead to artifacts if step sizes are not carefully clamped. Precision issues may arise with very small distances due to the non-rigid nature of the transform [S1, unofficial].

## Background
Butterfly Ray Marching with SDFs is a technique demonstrated in the project "manim-glsl-butterflies" by el-bakkali [S1, unofficial]. It utilizes GLSL shaders within the Manim animation engine to render a swarm of butterflies. The method combines SDFs for geometric representation with ray marching for rendering, specifically addressing the challenge of animating wing flapping through a local coordinate transformation.

## Sources
[S1] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
