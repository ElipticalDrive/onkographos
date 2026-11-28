# Domain Warped FBM

Also called: 2D Clouds, Domain Warping, FastNoiseLite Domain Warp, Inigo Quilez Domain Warping, NoiseTexture2D/3D Domain Warp

## Overview
Domain Warped FBM is a procedural noise technique where the output of a noise function is fed back as a coordinate offset to distort the sampling domain, creating organic, fluid-like patterns [S2]. It differs from standard FBM by introducing non-linear spatial distortion that mimics turbulent flow, and from simple noise offsets by using the noise amplitude itself to drive the displacement [S2].

## How it works
**Inputs and Outputs**
- **Input**: Base 2D or 3D coordinate `p` (or `uv`), time `t` (optional, for animation), and warp strength parameter `WARP_STRENGTH`.
- **Output**: A scalar noise value `f` and, optionally, normal vectors for lighting.

**Data Structures**
- Coordinates are represented as 2D vectors (`vec2`) or 3D vectors (`vec3`) [S2].
- Noise functions `fbm` return a scalar in `[0, 1]` or `[-1, 1]` depending on implementation; the examples below assume a range that is normalized to `[0, 1]` via `0.5 + 0.5 * noise` [S2].

**Steps**
1. **Compute Primary Warp Field**: Calculate a displacement vector `q` using a base FBM function on the input coordinates.
   $$ q = \text{fbm}(p \cdot 0.5) $$
   [S2]

2. **Compute Secondary Warp Field**: Calculate a second displacement vector `r` using FBM on coordinates offset by `q` scaled by `WARP_STRENGTH`.
   $$ r = \text{fbm}(p + \text{WARP\_STRENGTH} \cdot q + \text{offset}) $$
   [S2]
   *Note: An `offset` vector (e.g., `vec2(1.7, 9.2)`) is often added to prevent symmetry artifacts [S2].*

3. **Compute Final Noise Value**: Sample the FBM function again using coordinates warped by `r`.
   $$ f = 0.5 + 0.5 \cdot \text{fbm}(p + \text{WARP\_STRENGTH} \cdot r) $$
   [S2]

4. **Modulate Output**: Mix the final noise value `f` with a cubic function of `f`, weighted by the magnitude of the secondary warp `r`.
   $$ f_{\text{final}} = \text{mix}(f, f^3 \cdot 3.5, f \cdot \|r\|) $$
   [S2]
   *Note: In some variants, the weight uses `f \cdot |n.x|` where `n` is a component of the warp field [S2].*

5. **Compute Normals (Optional)**: Approximate derivatives using finite differences with epsilon `eps` to compute normals for lighting.
   $$ n_x = \text{fbm}(p + \text{eps} \cdot \hat{x} + \text{WARP\_STRENGTH} \cdot r(p + \text{eps} \cdot \hat{x})) $$
   $$ n_y = \text{fbm}(p + \text{eps} \cdot \hat{y} + \text{WARP\_STRENGTH} \cdot r(p + \text{eps} \cdot \hat{y})) $$
   $$ \text{normal} = \text{normalize}(n_x - f, \epsilon, n_y - f) $$
   [S2]

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def domain_warped_fbm(p, time=0.0, warp_strength=0.5, eps=1e-4):
    """
    Computes domain warped FBM.
    p: vec2 or vec3 coordinate
    time: float for animation
    warp_strength: float scaling the warp displacement
    eps: float for finite difference normal calculation
    """
    # Step 1: Primary warp field q
    # fbm2 returns a vector of same dimension as input
    q = fbm2(p * 0.5 + vec2(time * 0.1, 0.0))
    
    # Step 2: Secondary warp field r
    # Offset vector to break symmetry
    offset = vec2(1.7, 9.2)
    r = fbm2(p + warp_strength * q + offset)
    
    # Step 3: Final noise value f
    f = 0.5 + 0.5 * fbm(p + warp_strength * r)
    
    # Step 4: Modulate f
    # Mix f with f^3 * 3.5 weighted by f * length(r)
    weight = f * length(r)
    f_final = mix(f, f * f * f * 3.5, weight)
    
    # Step 5: Compute normals (optional, for lighting)
    # Calculate perturbed noise values for x and y derivatives
    eps_x = vec2(eps, 0.0)
    eps_y = vec2(0.0, eps)
    
    # Recompute r for perturbed positions to maintain consistency
    r_x = fbm2(p + eps_x + warp_strength * q + offset)
    r_y = fbm2(p + eps_y + warp_strength * q + offset)
    
    f_x = 0.5 + 0.5 * fbm(p + eps_x + warp_strength * r_x)
    f_y = 0.5 + 0.5 * fbm(p + eps_y + warp_strength * r_y)
    
    # Normal vector approximation
    normal = normalize(vec3(f_x - f, eps, f_y - f))
    
    return f_final, normal

def fbm2(p):
    """
    Placeholder for 2D FBM function.
    Returns a vec2.
    """
    # Implementation depends on underlying noise generator (e.g., Simplex, Perlin)
    # Typically sums multiple octaves
    return vec2(fbm_scalar(p), fbm_scalar(p + vec2(1.0)))

def fbm_scalar(p):
    """
    Placeholder for scalar FBM function.
    Returns a float in [0, 1].
    """
    # Implementation depends on underlying noise generator
    return 0.5 + 0.5 * noise(p)

def noise(p):
    """
    Placeholder for base noise function (e.g., Simplex, Perlin).
    Returns a float in [-1, 1].
    """
    # Implementation depends on underlying noise generator
    return 0.0
```

**Parameters**
- `WARP_STRENGTH`: Controls the intensity of the distortion. Typical values are around `0.5` [S2].
- `eps`: Finite difference step size for normal calculation. Typically `1.0 / resolution` [S2].

**Edge Cases and Pitfalls**
- **Precision**: Domain warping can cause coordinate values to grow large, potentially leading to precision issues in the underlying noise function. Using `fbm2` (returning vectors) helps maintain structure [S2].
- **Symmetry**: Without an offset vector in the secondary warp, patterns may exhibit unwanted symmetry. An offset like `vec2(1.7, 9.2)` is recommended [S2].
- **Performance**: Computing normals requires additional noise evaluations, increasing computational cost.

## Subtopics

### [FBM Combinations](3962-fbm-combinations.md)
Combines multiple FBM variants for complex textures. Choose when you need to blend different noise characteristics.

### [Terrain Heightmap Generation via Noise](4214-terrain-heightmap-generation-via-noise.md)
Uses noise for terrain elevation. Choose for landscape generation.

## Operating characteristics
- **Time Complexity**: O(1) per sample, but with a higher constant factor due to multiple noise evaluations (typically 3-4 FBM calls per sample) [S2].
- **Memory Complexity**: O(1) as it is a procedural function.
- **Performance**: On a NoiseTexture2D sprite, applying domain warp reduces maximum error from 143/255 to 1-3/255 compared to a baseline, while keeping the whole-frame mean error stable at 0.72/255 [S1, unofficial].
- **Strengths**: Produces highly organic, fluid-like patterns suitable for clouds, smoke, and lava [S2].
- **Limitations**: Computationally more expensive than standard FBM. Can introduce artifacts if warp strength is too high.

## Background
Domain Warping was popularized by Inigo Quilez in his shader tutorials [S2]. It is implemented in libraries like FastNoiseLite, where it is exposed as a domain warp feature in `NoiseTexture2D` and `NoiseTexture3D` [S1, unofficial]. The technique is used in real-time rendering for procedural textures and effects.

## Sources
[S1] feat(renderer): draw FogVolume and FogMaterial into the volumetric fog, https://github.com/Vortiago/textscene-inspector/pull/732, fetched 2026-10-26 (unofficial)
[S2] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
