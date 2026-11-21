# Skybox Rendering with Procedural Shading

Also called: Procedural Sky, Shader-based Sky

## Overview
A procedural sky rendered on a back-face culled sphere using a custom shader material, combining horizon/zenith gradients, ground reflection, tiling noise clouds, and a sun disk with glow [S1, unofficial]. It differs from kernel-based forward rendering by computing the background color per-fragment via analytical functions and noise sampling rather than precomputing or accumulating contributions in a separate pass [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: Camera view direction vector $\mathbf{dir}$ (normalized).
- Input: Sun direction vector $\mathbf{sunDir}$ (normalized).
- Input: Cloud texture (2D noise map).
- Input: Color constants for horizon, zenith, ground, and sun.
- Output: Final background color $\mathbf{col}$.

**Data structures:**
- Sphere geometry: Radius $R = 440$ units. Resolution: 32x16 segments.
- Cloud texture: 2D noise texture, sampled via bilinear interpolation.

**Parameters:**
- Horizon blend exponent: $0.55$
- Ground blend threshold: $0.25$ (smoothstep range start)
- Cloud UV scale: $0.22$
- Cloud UV offset: $(0.31, 0.17)$
- Cloud noise threshold: $0.5$ to $0.78$
- Cloud vertical fade: $0.02$ to $0.22$
- Cloud opacity multiplier: $0.85$
- Cloud color mix threshold: $0.55$ to $0.9$
- Sun glow exponents: $12$ (weight $0.25$), $220$ (weight $0.6$)
- Sun disk smoothstep: $0.99935$ to $0.9997$
- Sun disk intensity multiplier: $18$
- Cloud masking factor for sun: $0.8$

**Steps:**
1. Calculate normalized camera direction $\mathbf{dir}$.
2. Compute vertical component $y = \mathbf{dir}.y$.
3. Determine sun direction $\mathbf{sunDir}$.
4. Calculate horizon blend factor $up = \max(y, 0)$.
5. Mix horizon and zenith colors: $\mathbf{col} = \text{mix}(\text{color}(\text{HORIZON}), \text{color}(\text{ZENITH}), \text{pow}(up, 0.55))$.
6. Blend with ground color based on negative $y$: $\mathbf{col} = \text{mix}(\mathbf{col}, \text{color}(\text{GROUND}), \text{smoothstep}(0, 0.25, -y))$.
7. Generate cloud layer by projecting direction onto a high plane. Calculate UVs: $\mathbf{cuv} = (\mathbf{dir}.xz / (y + 0.12)) \times 0.22 + (0.31, 0.17)$.
8. Sample tiling noise texture $n = \text{texture}(\text{cloudTexture}, \mathbf{cuv}).r$.
9. Compute cloud opacity: $\mathbf{cloud} = \text{smoothstep}(0.5, 0.78, n) \times \text{smoothstep}(0.02, 0.22, y) \times 0.85$.
10. Compute cloud color based on noise: $\mathbf{cloudCol} = \text{mix}((0.82, 0.86, 0.93), (1.15, 1.15, 1.15), \text{smoothstep}(0.55, 0.9, n))$.
11. Blend cloud into sky: $\mathbf{col} = \text{mix}(\mathbf{col}, \mathbf{cloudCol}, \mathbf{cloud})$.
12. Render sun disk and glow. Calculate dot product $s = \max(\mathbf{dir} \cdot \mathbf{sunDir}, 0)$.
13. Compute glow using power functions: $\mathbf{glow} = \text{pow}(s, 12) \times 0.25 + \text{pow}(s, 220) \times 0.6$.
14. Compute disk using smoothstep threshold: $\mathbf{disk} = \text{smoothstep}(0.99935, 0.9997, s) \times (1 - \mathbf{cloud} \times 0.8) \times 18$.
15. Add sun color to sky: $\mathbf{col} = \mathbf{col} + \text{color}(\text{SUN\_COLOR}) \times (\mathbf{glow} + \mathbf{disk})$.

**Pitfalls:**
- Sun disk must be masked by clouds to avoid rendering through them. Implemented by multiplying disk intensity by $1 - \mathbf{cloud} \times 0.8$ [S1, unofficial].
- Precision issues may arise near the horizon due to the $y+0.12$ term in cloud UV calculation; ensure $y$ does not cause division by zero or extreme UV stretching [G].
- The sphere must be rendered with depth write disabled and behind all other geometry (render order -1000) to avoid z-fighting with foreground objects [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_skybox(dir: Vec3, sunDir: Vec3, cloudTexture: Texture2D) -> Vec3:
    # Constants
    HORIZON_COLOR = Vec3(0.5, 0.7, 0.9)
    ZENITH_COLOR = Vec3(0.1, 0.3, 0.6)
    GROUND_COLOR = Vec3(0.4, 0.3, 0.2)
    SUN_COLOR = Vec3(1.0, 0.9, 0.7)
    
    # Step 1-3: Basic setup
    y = dir.y
    up = max(y, 0.0)
    
    # Step 4-5: Sky gradient
    col = mix(HORIZON_COLOR, ZENITH_COLOR, pow(up, 0.55))
    
    # Step 6: Ground blend
    ground_blend = smoothstep(0.0, 0.25, -y)
    col = mix(col, GROUND_COLOR, ground_blend)
    
    # Step 7-8: Cloud UVs and sampling
    # Avoid division by zero by adding small epsilon
    uv_x = dir.x / (y + 0.12) * 0.22 + 0.31
    uv_y = dir.z / (y + 0.12) * 0.22 + 0.17
    cuv = Vec2(uv_x, uv_y)
    
    # Sample noise texture (assuming normalized coordinates)
    n = cloudTexture.sample(cuv).r
    
    # Step 9: Cloud opacity
    cloud_alpha = smoothstep(0.5, 0.78, n) * smoothstep(0.02, 0.22, y) * 0.85
    
    # Step 10: Cloud color
    cloud_col = mix(Vec3(0.82, 0.86, 0.93), Vec3(1.15, 1.15, 1.15), smoothstep(0.55, 0.9, n))
    
    # Step 11: Blend clouds
    col = mix(col, cloud_col, cloud_alpha)
    
    # Step 12-15: Sun disk and glow
    s = max(dot(dir, sunDir), 0.0)
    
    # Glow
    glow = pow(s, 12.0) * 0.25 + pow(s, 220.0) * 0.6
    
    # Disk with cloud masking
    disk = smoothstep(0.99935, 0.9997, s) * (1.0 - cloud_alpha * 0.8) * 18.0
    
    # Add sun
    col = col + SUN_COLOR * (glow + disk)
    
    return col
```

## Subtopics

### [Background Rendering](5042-background-rendering.md)
Procedural background with sky gradient, sun glow, volumetric light shafts, and vignette.

### [Background Rendering Kernel (Forward)](14913-background-rendering-kernel-forward.md)
Kernel-based forward rendering approach.

## Operating characteristics
- Time complexity: $O(1)$ per fragment, independent of scene complexity.
- Memory complexity: $O(1)$ for sphere geometry, plus texture memory for cloud noise.
- Performance: Fast, as it avoids complex ray marching or volumetric integration.
- Strengths: Simple to implement, visually appealing for stylized or low-poly scenes, no need for complex lighting calculations.
- Limitations: Lacks physical accuracy, no atmospheric scattering, limited detail compared to volumetric methods.
- When to use: Stylized games, low-end hardware, simple scenes.
- When not to use: Realistic simulations, complex atmospheric effects.

## Background
Originated in early 3D graphics for efficient sky rendering. Popularized by voxel-based games and indie titles for its simplicity and visual appeal. Used in games like Voxel Wars for its lightweight nature and customizable appearance.

## Sources
[S1] sh1ftmaker/voxel-wars, https://github.com/sh1ftmaker/voxel-wars, fetched 2026-10-28 (unofficial)
