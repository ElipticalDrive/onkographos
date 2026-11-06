# Local Cloud Flow with FlowMap

Also called: FlowMap Cloud Movement, Noise Texture Flow Offset

## Overview
Local Cloud Flow with FlowMap simulates small-scale, local turbulence and swirls in volumetric clouds by offsetting noise sampling coordinates using a pre-computed flow direction texture [S1, unofficial]. It addresses the problem of static or unnaturally drifting cloud details by introducing spatially varying wind vectors, distinct from global advection which handles large-scale directional movement [S1, unofficial]. It differs from [Global Cloud Flow](2270-global-cloud-flow.md) by using per-pixel texture lookups for flow direction rather than global vector parameters, making it suitable for scenes where local turbulence is critical [S1, unofficial].

## How it works
**Inputs:**
- `FlowMap`: A 2D texture storing 3D flow direction vectors [S1, unofficial].
- `ShapeNoise`: A 3D noise texture defining the cloud's base density structure [S1, unofficial].
- `time`: A continuous time variable [S1, unofficial].
- `worldPos`: The 3D world position of the current sample point [S1, unofficial].

**Outputs:**
- `density`: The final noise value at the sample point, incorporating local flow [S1, unofficial].

**Data Structures:**
- `FlowMap`: Stored as a standard 2D texture. Each texel contains a 3D vector representing the local flow direction [S1, unofficial][G].
- `ShapeNoise`: 3D texture sampled at coordinates $(x, y, z)$ [G].

**Steps:**
1. **Calculate UV Coordinates:** Extract the X and Z components of the world position to form 2D UV coordinates for the FlowMap [S1, unofficial].
   $$ uv = (worldPos.x, worldPos.z) $$
2. **Sample Flow Direction:** Sample the `FlowMap` at `uv` to obtain the local flow direction vector $\vec{f}$ [S1, unofficial].
   $$ \vec{f} = \text{Sample}(FlowMap, uv) $$
3. **Compute Flow Offsets:** To prevent severe stretching of noise textures over time due to varying flow speeds and directions, use the fractional part of time to create a repeating flow cycle [S1, unofficial].
   - Calculate the primary time offset: $t_1 = \text{fract}(time)$ [S1, unofficial].
   - Calculate the secondary time offset (half-beat delay): $t_2 = \text{fract}(time + 0.5)$ [S1, unofficial].
   - Compute the spatial offsets for noise sampling:
     $$ \vec{offset}_1 = \vec{f} \cdot t_1 $$
     $$ \vec{offset}_2 = \vec{f} \cdot t_2 $$
4. **Sample Noise Textures:** Sample the `ShapeNoise` texture twice using the base world position offset by the calculated flow vectors [S1, unofficial].
   $$ n_1 = \text{Sample}(ShapeNoise, worldPos + \vec{offset}_1) $$
   $$ n_2 = \text{Sample}(ShapeNoise, worldPos + \vec{offset}_2) $$
5. **Interpolate:** Interpolate between the two noise samples to achieve a smooth flow effect [S1, unofficial].
   $$ density = \text{lerp}(n_1, n_2, \alpha) $$

**Parameters:**
- `time`: Continuous time variable [S1, unofficial].
- `fract()`: Returns the fractional part of a number [G].

**Edge Cases and Pitfalls:**
- **Stretching:** Without the `fract(time)` mechanism, noise textures stretch severely over time due to varying flow directions and speeds [S1, unofficial].
- **Performance:** Applying FlowMap to all noise textures is expensive. It is typically applied only to `ShapeNoise`, while other textures remain unchanged [S1, unofficial].
- **Y-Axis Flow:** The Y component of the flow vector can control vertical rolling of the noise, affecting the overall cloud distribution over time [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_cloud_flow(world_pos, time, flow_map, shape_noise):
    # Step 1: Calculate UV from XZ world coordinates
    uv = (world_pos.x, world_pos.z)
    
    # Step 2: Sample flow direction
    flow_dir = flow_map.sample(uv)
    
    # Step 3: Compute time-based offsets to prevent stretching
    t1 = fract(time)
    t2 = fract(time + 0.5)
    
    offset1 = flow_dir * t1
    offset2 = flow_dir * t2
    
    # Step 4: Sample shape noise twice
    noise1 = shape_noise.sample(world_pos + offset1)
    noise2 = shape_noise.sample(world_pos + offset2)
    
    # Step 5: Interpolate for smooth flow
    # Blend factor calculation is implementation-specific but typically 
    # relates to the progression within the fract(time) cycle [G].
    blend = fract(time) 
    density = lerp(noise1, noise2, blend)
    
    return density
```

## Subtopics

### [Global Cloud Flow](2270-global-cloud-flow.md)
Simulates large-scale, directional movement using global vector parameters. Choose when local turbulence is less critical than global advection.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Applying FlowMap to all noise textures is expensive; limiting it to `ShapeNoise` reduces overhead [S1, unofficial].
- **Strengths:** Provides realistic local turbulence and swirls without complex vector field calculations at runtime [S1, unofficial].
- **Limitations:** Prone to noise stretching if not handled with time-fractionation; requires pre-computed flow data [S1, unofficial].

## Background
Originating from practical implementations in real-time rendering, such as those described in industry articles on volumetric cloud effects [S1, unofficial]. It is used in games and simulations to add detail to cloud volumes, often in conjunction with global flow and lighting models like Beer's Law for transmittance [S1, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
