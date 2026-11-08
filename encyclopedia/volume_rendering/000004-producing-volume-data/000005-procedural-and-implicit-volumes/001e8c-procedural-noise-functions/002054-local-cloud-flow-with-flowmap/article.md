# Local Cloud Flow with FlowMap

Also called: FlowMap Cloud Movement, Noise Texture Flow Offset

## Overview
Local Cloud Flow with FlowMap simulates local cloud movement, such as swirls and turbulence, by offsetting noise texture sampling coordinates using a pre-computed FlowMap texture that stores flow direction [S1, unofficial]. It differs from [Global Cloud Flow](2270-global-cloud-flow.md) by applying per-pixel, spatially varying vector offsets derived from a texture lookup rather than uniform global wind vectors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** World space position (specifically XZ components), time parameter ($t$), a FlowMap texture storing 3D flow vectors, and a Shape Noise texture.
*   **Outputs:** Displaced noise sampling coordinates for the Shape Noise texture.

**Data Structures:**
*   **FlowMap:** A 2D texture where each texel stores a 3D vector $(u, v, w)$ representing the local flow direction and magnitude.
*   **Shape Noise:** A procedural or texture-based noise field used to define the cloud's base structure.

**Steps:**
1.  **Calculate UV Coordinates:** Derive 2D UV coordinates from the world space XZ components of the current sampling position.
2.  **Sample Flow Direction:** Sample the FlowMap texture at the calculated UV coordinates to retrieve a 3D flow direction vector $\vec{F}$.
3.  **Compute Primary Offset:** Offset the noise texture sampling coordinates by the flow direction scaled by time. To prevent severe stretching of the noise texture over time due to varying flow directions and speeds, the time component is wrapped using the fractional part: $t_{primary} = \text{fract}(t)$ [S1, unofficial].
4.  **Compute Secondary Offset:** Sample the noise texture again using a secondary time offset to capture the flow state half a "beat" later. The time parameter is calculated as $t_{secondary} = \text{fract}(t + 0.5)$ [S1, unofficial].
5.  **Interpolate:** Interpolate between the noise values obtained from the primary and secondary samples to achieve a smooth, continuous flow effect [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_cloud_flow(world_pos_xz, time, flow_map, shape_noise):
    # 1. Calculate UV coordinates from world XZ
    uv = world_pos_xz
    
    # 2. Sample FlowMap to get 3D flow direction
    flow_direction = flow_map.sample(uv)
    
    # 3. Primary sample: offset by fract(time)
    # Note: The exact scaling of flow_direction to match noise frequency is implementation-dependent [G]
    offset_primary = flow_direction * fract(time)
    sample_pos_primary = uv + offset_primary
    noise_val_primary = shape_noise.sample(sample_pos_primary)
    
    # 4. Secondary sample: offset by fract(time + 0.5)
    offset_secondary = flow_direction * fract(time + 0.5)
    sample_pos_secondary = uv + offset_secondary
    noise_val_secondary = shape_noise.sample(sample_pos_secondary)
    
    # 5. Interpolate for smooth flow
    # The interpolation factor is not specified in sources [G]
    # Typically a smoothstep or linear lerp based on the fractional time component
    flow_factor = fract(time) 
    final_noise = lerp(noise_val_primary, noise_val_secondary, flow_factor)
    
    return final_noise
```

**Parameters:**
*   **Time Step Offset:** The secondary sample is offset by 0.5 relative to the primary sample in the fractional time domain [S1, unofficial].
*   **FlowMap Application Scope:** The FlowMap is applied only to the Shape Noise texture. Other noise textures (e.g., detail noise) remain unchanged to reduce computational cost [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Noise Stretching:** Without the `fract(time)` wrapping, noise textures stretch severely over time due to varying flow directions and speeds in the FlowMap [S1, unofficial].
*   **Performance:** Applying the FlowMap logic to all noise textures is expensive; it is restricted to Shape Noise to maintain performance [S1, unofficial].

## Subtopics

### Noise Texture Advection
Noise Texture Advection directly advects the noise input texture using given vector fields via simple Euler integration, rather than calculating streamlines at each position [S2, book]. It is performed in parallel for each pixel on the GPU [S2, book].

*   **Definition:** Direct advection of noise input textures using vector fields via Euler integration [S2, book].
*   **Steps:**
    1.  Initialize $p_0^\lambda = \beta_{x,y}$ (input noise value at pixel $(x,y)$) for each eigenvector field $\lambda \in \{\lambda_1, \lambda_2\}$ [S2, book].
    2.  Iterate: $p_{i+1}^\lambda = k \cdot \beta_{x,y} + (1-k) \cdot \frac{f_{p_i^\lambda}((x,y) + v'_\lambda) + f_{p_i^\lambda}((x,y) - v'_\lambda)}{2}$ [S2, book].
    3.  Stop when the value change exceeds a threshold [S2, book].
*   **Formula:** Input field definition: $f_p : (x, y) \rightarrow p$, with $x, y, p \in [0, 1]$. Continuity is ensured with interpolating values in-between [S2, book].
*   **Parameters:**
    *   **Directionality:** Eigenvectors $v'_\lambda$ do not have an orientation, so advection must be done in both directions ($+v'_\lambda$ and $-v'_\lambda$) [S2, book].
    *   **Iteration Limit:** The iteration stops when the value change exceeds a threshold [S2, book].
*   **Layout:** Separate input and output fields $p_i$ are used for each eigenvector field. After 10 iterations, the red channel contains $p_{\lambda_1}^9$ and the green channel contains $p_{\lambda_2}^9$ [S2, book].

## Operating characteristics
*   **Performance:** Applying FlowMap to all noise textures is computationally expensive due to the double sampling and interpolation required per noise layer. It is optimized by applying it only to Shape Noise [S1, unofficial].
*   **Visual Quality:** The `fract(time)` technique effectively prevents the severe stretching artifacts that occur with continuous flow offsets, providing a stable, looping local turbulence effect [S1, unofficial].
*   **Limitations:** The method relies on a pre-computed FlowMap texture, which adds memory overhead and requires careful authoring of flow directions to avoid unnatural artifacts [S1, unofficial].

## Background
Local Cloud Flow with FlowMap is a technique used in real-time volume cloud rendering, particularly in game engines, to simulate complex local cloud movements like swirls. It builds on the principles of procedural noise functions by introducing spatially varying domain offsets [S1, unofficial]. The specific implementation details, including the `fract(time)` smoothing technique, are documented in industry articles discussing volume cloud implementation in games [S1, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
