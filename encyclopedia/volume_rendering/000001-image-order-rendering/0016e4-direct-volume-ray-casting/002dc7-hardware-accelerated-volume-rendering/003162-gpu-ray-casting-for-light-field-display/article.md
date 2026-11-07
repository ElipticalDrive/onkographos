# GPU Ray Casting for Light Field Display

Also called: Direct Volume Rendering on Light Field Display, Interactive Light Field Volume Rendering, MCOP Ray Casting with Adaptive Mipmap Level Selection, MCOP projection, Multi-viewer 3D Volume Rendering

## Overview
This method performs interactive direct volume rendering on a multi-projector light field display by computing per-pixel rays from projector positions through screen pixels into the volume using Multi-Camera Optical Projection (MCOP) [S1]. It differs from standard GPU ray casting by replacing fixed-function vertex interpolation with a coarse proxy geometry and vertex-shader-based MCOP projection to prevent ray parameter distortion, and by employing depth-dependent mipmap level selection to match sampling rates to the display's spatial resolution [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Volume data $V$ stored in a 4-component 3D texture (`sampler3D`). The RGB channels contain precomputed unscaled gradients; the A channel contains the scalar voxel value [S1].
    *   A 1D RGBA lookup table (`sampler1D`) for the transfer function [S1].
    *   Uniforms: Projector position $E_w$ (`float3`), virtual observer distance $V_d$ (`float`), virtual observer height $V_h$ (`float`), log2 voxel size $\log_2 V$ (`float`), step size $DL$ (`float`), screen pixel size $S_0$ (`float`), angular spread factor $\tan \Phi$ (`float`), and the inverse model-view matrix `mv_inv` (`float4x4`) [S1].
    *   Fragment coordinates $(x, y)$ representing the screen pixel position in the display reference frame [S1].
*   **Outputs:**
    *   Accumulated color $I$ and opacity $a$ passed to the frame buffer [S1].

**Data Structures**
The method uses two structures defined in the source code [S1]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```c
struct ray_t {
    float3 org; // Ray origin in texture coordinates
    float3 dir; // Normalized ray direction in texture coordinates
    float len;  // Integration length within the volume
};

struct fragment_out {
    float4 color; // Output color (RGB) and accumulated opacity (A)
};
```

**Algorithm Steps**
1.  **Proxy Geometry Setup:** A coarsely tessellated proxy geometry (8x8 quads) of a slightly enlarged bounding volume is used [S1]. This avoids the incorrect per-pixel ray parameter values caused by linear interpolation of vertex attributes in the fixed-function pipeline, which fails because MCOP cannot be recast into a traditional homogeneous matrix and produces curved lines [S1].
2.  **Ray Origin and Direction Computation:** In the vertex shader, the MCOP projection is performed. The fragment shader receives the unnormalized projected position $S$ (screen pixel position in the display reference frame) [S1]. The ray direction $d$ in world coordinates is computed using the emitter position $E$ and virtual viewer parameters $V$ [S1]:
    $$
    d = \left[ \frac{S_x - E_x}{E_z}, \frac{S_y - V_y}{V_z}, -1 \right]
    $$
    Note: The source code implementation uses specific uniform names $E_w, V_d, V_h$ to compute the direction vector components [S1].
3.  **Coordinate Transformation:** The infinite ray defined by origin $S$ and direction $d$ is transformed into local volume coordinates (coincident with texture coordinates) by applying the inverse model-view homogeneous transform matrix `mv_inv` [S1].
4.  **Ray Clipping and Validation:** The ray is clipped against the unit box (the volume domain in texture space) to compute the entry point and integration length [S1]. If the resulting ray length is null (or less than a small epsilon), the fragment is discarded as invalid [S1].
5.  **Mipmap Level Selection:** The volume is stored in a mipmap pyramid. At each sampling step, the depth $z$ of the current sample is computed by transforming its texture-space position to world space using a dot product with the model-view matrix [S1]:
    $$
    z = | \mathbf{m}_{0,2} \cdot \mathbf{p}_{tex} |
    $$
    where $\mathbf{m}_{0,2}$ is the third column of the model-view matrix and $\mathbf{p}_{tex}$ is the homogeneous position of the sample [S1]. The mipmap level $l$ is chosen as the coarsest level with a sufficiently small voxel size, calculated as [S1]:
    $$
    l = \text{median}(0, L_{max}, \lfloor \log_2 \frac{s(z)}{v_0} \rfloor)
    $$
    where $s(z)$ is the variable step-size, $L_{max}$ is the maximum mipmap level, and $v_0$ is the finest voxel size [S1]. The source code approximates this with:
    $$
    l = \log_2(S_0 + 2 \tan \Phi \cdot z) - \log_2 V
    $$
6.  **Ray Marching Loop:** The ray is stepped through the volume. At each step:
    *   The texture is sampled at the current position `uvw` using the computed mipmap level $l$ via `tex3Dlod` (or `mipmap_value`) [S1].
    *   The scalar value is mapped to color $C_i$ and opacity $\alpha_i$ using the transfer function and shading model (implementation details omitted in source, marked as placeholder) [S1].
    *   Front-to-back compositing updates the accumulated color $I$ and opacity $a$:
        $$
        I = I + (1.0 - a) C_i
        $$
        $$
        a = a + (1.0 - a) \alpha_i
        $$
    *   The position is stepped along the ray direction by $DL$ [S1].
    *   The loop terminates if full opacity is reached or the volume is exited [S1].

**Pseudocode**
The following is a self-contained fragment shader implementation based on the source [S1]. Helper functions for geometric clipping and transfer functions are assumed or implemented as standard operations.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Uniforms
# Ew: Projector position (float3)
# Vd: Virtual observer distance (float)
# Vh: Virtual observer height (float)
# log2V: Log2 of voxel size (float)
# DL: Step size (float)
# S0: Screen pixel size (float)
# tanPhi: Angular spread factor (float)
# mv_inv: Inverse model-view matrix (float4x4)
# mv_modelview: Model-view matrix (float4x4)
# volume_tex: 3D texture sampler
# lut_tex: 1D texture sampler

struct Ray {
    float3 org;
    float3 dir;
    float len;
};

// Helper: Clip ray against unit box [0,1]^3
// Returns {tmin, tmax} intersection distances
function clip_to_unit_box(origin, direction):
    tmin = 0.0
    tmax = 1.0
    for axis in [0, 1, 2]:
        if direction[axis] == 0:
            if origin[axis] < 0 or origin[axis] > 1:
                return {0, 0} // Parallel and outside
        else:
            t1 = (0 - origin[axis]) / direction[axis]
            t2 = (1 - origin[axis]) / direction[axis]
            t_enter = min(t1, t2)
            t_exit = max(t1, t2)
            tmin = max(tmin, t_enter)
            tmax = min(tmax, t_exit)
    if tmin > tmax:
        return {0, 0}
    return {tmin, tmax}

// Helper: Compute mipmap level
function compute_mipmap_level(uvw, mv_modelview):
    // Compute depth z from screen
    // glstate.matrix.modelview[0][2] corresponds to the z-row of the modelview matrix
    // Using the third column (index 2) of the model-view matrix for the z-component transformation
    z = abs(dot(mv_modelview[2], float4(uvw, 1.0)))
    // Equation from source: l = log2(S0 + 2*tanPhi*z) - log2V
    l = log2(S0 + 2 * tanPhi * z) - log2V
    return l

// Helper: Apply transfer function and shading
// Returns {color, alpha}
function apply_transfer_function_and_shading(value, gradient):
    // Implementation details omitted in source [S1]
    // Typically involves looking up value in lut_tex and using gradient for Phong shading
    color = texture(lut_tex, value).rgb
    alpha = texture(lut_tex, value).a
    return {color, alpha}

function fragment_shader(x, y):
    // 1. Compute ray in world coordinates
    ray_o_world = float4(x, y, 0, 0)
    // Direction calculation based on MCOP projection parameters
    // Source: d = [-(Ew.x-x)/Ew.z, (Vh-y)/Vd, -1]
    ray_d_world = float4(-(Ew.x - x) / Ew.z, (Vh - y) / Vd, -1, 0)

    // 2. Transform to texture coordinates
    tex_o = mul(mv_inv, ray_o_world).xyz
    tex_e = mul(mv_inv, ray_o_world + ray_d_world).xyz

    // 3. Clip to unit box
    dir = normalize(tex_e - tex_o)
    tmin_tmax = clip_to_unit_box(tex_o, dir)
    
    // 4. Validate ray length
    if tmin_tmax.y - tmin_tmax.x < 1e-6:
        discard // Out of bounds

    ray_org = tex_o + dir * tmin_tmax.x
    ray_len = tmin_tmax.y - tmin_tmax.x
    ray_dir = dir

    // 5. Initialize compositing
    delta_dir = ray_dir * DL
    N = int(ray_len / DL)
    uvw = ray_org
    I = float3(0, 0, 0)
    a = 0.0
    a_max = 0.95 // Typical early termination threshold [G]

    // 6. Ray marching loop
    for i from 0 to N-1:
        if a >= a_max:
            break
        
        // Compute mipmap level based on depth
        l = compute_mipmap_level(uvw, mv_modelview)
        
        // Sample volume texture
        // Source uses tex3Dlod or mipmap_value(uvw, l)
        // Volume layout: RGB=gradient, A=voxel value
        tex_sample = texture(volume_tex, uvw, l)
        voxel_value = tex_sample.a
        gradient = tex_sample.rgb
        
        // Apply transfer function and shading
        // Note: Source code leaves C_i and a_i undefined
        C_i, alpha_i = apply_transfer_function_and_shading(voxel_value, gradient)
        
        // Front-to-back compositing
        I = I + (1.0 - a) * C_i
        a = a + (1.0 - a) * alpha_i
        
        // Step along ray
        uvw += delta_dir

    return {color: float4(I, a)}
```

**Edge Cases and Pitfalls**
*   **Vertex Interpolation Error:** Standard fixed-function pipeline interpolation produces incorrect per-pixel ray parameters for MCOP. This is mitigated by the 8x8 proxy geometry and vertex-shader projection [S1].
*   **Tile Misalignment:** If the object moves too slowly relative to the display refresh rate, misalignment artifacts appear between tiles because the 96 projectors are updated in batches of 16 [S1].
*   **Ray Length Validation:** If the clipped ray length is less than epsilon, the fragment is discarded to avoid processing out-of-bounds rays [S1].
*   **Sampling Rate:** To maintain interactive frame rates, the sampling rate is often lowered (reduced pixel count, doubled step size) during interaction, which may reduce visual fidelity [S1].

## Subtopics

### MCOP Projection
A projection model for light field displays that provides a fully continuous perceived image, unlike discrete multiview technologies that cause abrupt view changes [S1].

### Adaptive Mipmap Sampling
A technique where the mipmap level is selected based on the depth $z$ from the screen and the angular spread $\tan \Phi$, ensuring the voxel size matches the spatial resolution at that depth [S1].

## Operating characteristics
*   **Time Complexity:** The cost is proportional to the number of rays (pixels) times the number of samples per ray. The number of samples is determined by the ray length divided by the step size $DL$ [S1].
*   **Memory Complexity:** Requires storage for the 3D volume texture (4 components) and the 1D lookup table [S1].
*   **Performance:** The main limiting factor is the single GPU volume renderer performance, as millions of rays may need to be propagated for hundreds of steps at full resolution [S1]. Interactive frame rates are achieved by lowering the sampling rate (reducing pixel count and increasing step size) [S1].
*   **Strengths:** Provides continuous parallax and multi-user capability; correct specular highlights follow the viewer's viewpoint; good registration between object space and physical space [S1].
*   **Limitations:** Performance is constrained by GPU throughput; tile misalignment artifacts can occur with slow refresh rates; requires careful tuning of sampling rates for interactivity [S1].
*   **When to use:** Interactive visualization on multi-projector light field displays where continuous viewing zones and multi-user interaction are required [S1].
*   **When not to use:** When high-resolution static rendering is required without interactivity, or when hardware does not support the required texture sampling operations (e.g., `tex3Dlod`) [S1].

## Background
The method was introduced by Agus et al. in 2008 [S1]. It builds upon the principles of direct volume ray casting and GPU acceleration, specifically addressing the challenges of rendering on interactive light field displays. The work demonstrates the rendering of datasets such as the Chameleon, Visible Human head, and rotational angiography using shaded volume rendering and maximum intensity projection [S1]. The approach contrasts with earlier multiview technologies that restricted users to fixed positions due to discrete viewing zones [S1].

## Sources
[S1] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
