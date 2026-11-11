# Local Cloud Flow with FlowMap

Also called: FlowMap Cloud Movement, Noise Texture Flow Offset

## Overview
Local Cloud Flow with FlowMap simulates localized cloud movement, such as swirls and eddies, by using a FlowMap texture to store flow directions [S1, unofficial]. It differs from Global Cloud Flow, which applies uniform vector offsets, by allowing per-pixel, spatially varying flow vectors derived from a texture lookup [S1, unofficial]. This method is primarily applied to Shape Noise to define the large-scale structure of clouds, while other noise textures (e.g., detail noise) remain static to save performance [S1, unofficial].

## How it works
**Inputs:**
- `time`: A floating-point time variable.
- `worldPos`: The 3D world position of the raymarching sample point.
- `flowMap`: A 2D texture storing 3D flow direction vectors (typically normalized).
- `noiseTexture`: The procedural or sampled noise texture (specifically Shape Noise).

**Outputs:**
- `finalNoiseValue`: The noise value at the advected coordinate.

**Data Structures:**
- `flowMap`: 2D texture, UV coordinates derived from world XZ. Stores 3D vectors $(u, v, w)$.
- `noiseTexture`: 3D or 2D texture, sampled at coordinates offset by the flow.

**Steps:**
1.  **Calculate UV Coordinates**: Extract the X and Z components of the world position to form 2D UV coordinates for the FlowMap.
    $$ \text{uv} = (\text{worldPos}.x, \text{worldPos}.z) $$
2.  **Sample Flow Direction**: Sample the FlowMap at `uv` to obtain a 3D flow direction vector $\mathbf{f}$.
    $$ \mathbf{f} = \text{Sample}(\text{flowMap}, \text{uv}) $$
3.  **Compute Time-Offset Coordinates**: To prevent severe stretching of noise features over time, the flow is repeated over a short distance using the fractional part of time.
    $$ t_1 = \text{fract}(\text{time}) $$
    $$ t_2 = \text{fract}(\text{time} + 0.5) $$
    The noise sampling coordinates are offset by the flow direction scaled by these time fractions. Let $\mathbf{P}$ be the original noise coordinate (derived from `worldPos`).
    $$ \mathbf{P}_1 = \mathbf{P} + \mathbf{f} \cdot t_1 $$
    $$ \mathbf{P}_2 = \mathbf{P} + \mathbf{f} \cdot t_2 $$
4.  **Sample Noise Textures**: Sample the Shape Noise texture at both offset coordinates.
    $$ n_1 = \text{Sample}(\text{noiseTexture}, \mathbf{P}_1) $$
    $$ n_2 = \text{Sample}(\text{noiseTexture}, \mathbf{P}_2) $$
5.  **Interpolate**: Interpolate between the two noise samples to achieve a smooth flow effect. The interpolation weight is typically determined by the fractional time step or a fixed blend factor.
    $$ \text{blend} = \text{fract}(\text{time} \cdot \text{speed}) $$
    $$ \text{finalNoiseValue} = \text{lerp}(n_1, n_2, \text{blend}) $$

**Parameters:**
- `time`: Controls the speed and phase of the flow.
- `flowMap` scale: Determines the spatial frequency of the flow directions.

**Edge Cases and Pitfalls:**
- **Stretching**: Without the `fract(time)` mechanism, noise textures stretch severely over time due to varying flow directions and speeds [S1, unofficial].
- **Performance**: Applying FlowMap to all noise textures is expensive. It is recommended to apply it only to Shape Noise, leaving other textures (like detail noise) unchanged [S1, unofficial].
- **Y-Axis Flow**: The Y component of the flow vector can control vertical rolling of the noise, affecting the overall cloud distribution over time [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_cloud_flow(time, world_pos, flow_map, noise_texture):
    # 1. Calculate UV from world XZ
    uv = vec2(world_pos.x, world_pos.z)
    
    # 2. Sample flow direction
    flow_dir = sample_3d_vector(flow_map, uv)
    
    # 3. Compute time offsets to prevent stretching
    t1 = fract(time)
    t2 = fract(time + 0.5)
    
    # 4. Calculate offset noise coordinates
    # Assuming base noise coordinate is derived from world_pos
    base_noise_coord = world_pos 
    p1 = base_noise_coord + flow_dir * t1
    p2 = base_noise_coord + flow_dir * t2
    
    # 5. Sample noise textures
    n1 = sample_noise(noise_texture, p1)
    n2 = sample_noise(noise_texture, p2)
    
    # 6. Interpolate for smooth flow
    # Blend factor can be derived from time or fixed
    blend = fract(time)
    final_value = lerp(n1, n2, blend)
    
    return final_value
```

## Subtopics

### [Global Cloud Flow](2270-global-cloud-flow.md)
Simulates global cloud movement by offsetting noise texture sampling coordinates over time using two global wind speeds. It differs from Local Cloud Flow with FlowMap by applying uniform vector offsets instead of per-pixel texture lookups.

### [Noise Texture Advection](10551-noise-texture-advection.md)
Directly advecting the noise input texture with given vector fields using simple Euler integration, rather than calculating streamlines at each position. This is performed in parallel for each pixel on the GPU.

## Operating characteristics
- **Performance**: Applying FlowMap to all noise textures is computationally expensive due to additional texture lookups and coordinate calculations. It is typically applied only to Shape Noise to balance quality and performance [S1, unofficial].
- **Visual Quality**: Provides realistic local swirls and eddies that uniform flow cannot achieve. The `fract(time)` technique prevents visual artifacts caused by noise stretching.
- **Limitations**: The method relies on a pre-defined FlowMap texture, which may not adapt dynamically to changing wind conditions without external updates.

## Background
This method is used in real-time volume rendering of clouds, particularly in game engines, to simulate local atmospheric dynamics. It builds on the principles of procedural noise functions by introducing spatially varying advection vectors [S1, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
