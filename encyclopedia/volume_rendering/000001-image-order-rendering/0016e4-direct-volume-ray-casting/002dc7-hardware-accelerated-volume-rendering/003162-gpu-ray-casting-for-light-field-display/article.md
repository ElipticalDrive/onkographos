# GPU Ray Casting for Light Field Display

Also called: Direct Volume Rendering on Light Field Display, Interactive Light Field Volume Rendering, MCOP Ray Casting with Adaptive Mipmap Level Selection, MCOP projection, Multi-viewer 3D Volume Rendering

## Overview
This method implements direct volume rendering on an interactive light field display by computing rays from projector positions through screen pixels into the volume using Multi-Camera Optical Projection (MCOP) [S1]. It solves the problem of incorrect per-pixel ray parameter values caused by standard vertex attribute interpolation in non-linear projections, ensuring a fully continuous perceived image for multiple viewers [S1]. Unlike traditional multiview technologies that force users into fixed positions due to abrupt view changes, this approach provides smooth parallax and accommodation cues [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Volume texture: A single 4-component 3D texture where RGB channels store the unscaled precomputed gradient and the A channel stores the voxel value [S1].
    *   LUT: A 1D RGBA texture containing an interactively modifiable look-up table [S1].
    *   Uniforms: Projector position $E_w$ (float3), Virtual observer distance $V_d$ (float), Virtual observer height $V_h$ (float), $\log_2 V$ (float, log2 of voxel size), $DL$ (float, step size), $S_0$ (float, screen pixel size), $\tan \Phi$ (float, angular spread factor) [S1].
    *   Matrices: OpenGL model-view matrix and its inverse [S1].
*   **Outputs:**
    *   Fragment color and opacity (float4) [S1].

**Data Structures:**
The method uses the following structures [S1]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct ray_t {
    float3 org;
    float3 dir;
    float len;
};
struct fragment_out {
    float4 color; // COLOR0
};
```

**Steps:**

1.  **Proxy Geometry Setup:** Render a coarsely tessellated proxy geometry (8x8 quads) of a slightly enlarged bounding volume [S1]. This avoids the interpolation errors inherent in standard fixed-function pipelines for MCOP projections [S1].
2.  **Vertex Shader Projection:** In the vertex shader, perform MCOP projection on the proxy vertices. Pass both normalized and unnormalized projected coordinates to the fragment stage [S1].
3.  **Ray Origin Computation:** In the fragment shader, receive the unnormalized projected position $S$ from the vertex stage. $S$ represents the screen pixel’s position in the display reference frame [S1].
4.  **Ray Direction Calculation:** Compute the ray direction $d$ passing through point $S$ using the MCOP projection parameterized by emitter $E$ and virtual viewer parameters $V$ [S1]:
    $$
    d = \left[ \frac{S_x - E_x}{E_z}, \frac{S_y - V_y}{V_z}, -1 \right]
    $$
    *Note: The source code in Appendix A of [S1] uses a simplified calculation for `ray_d` based on $E_w, V_d, V_h$ and pixel coordinates $x,y$:*
    $$
    \text{ray\_d} = \left[ -\frac{E_{w,x} - x}{E_{w,z}}, \frac{V_h - y}{V_d}, -1, 0 \right]
    $$
5.  **Coordinate Transformation:** Transform the infinite ray defined by $S$ and $d$ into local volume coordinates (coincident with texture coordinates) by applying the world-to-texture homogeneous transform matrix (inverse model-view matrix) [S1].
6.  **Ray Clipping and Validation:** Clip the line against the unit box to compute the ray entry point and integration lengths. If the ray length is null (or less than epsilon), discard the fragment as invalid [S1].
7.  **Ray Marching Loop:** Step through the ray in a loop, performing volume resampling and compositing until the volume is exited or full opacity is reached [S1].
8.  **Mipmap Level Selection:** At each sampling step, compute the depth $z$ of the current sample by transforming its position in texture space to world (screen-aligned) space using a single dot product [S1]. Compute the mipmap level $l$ using the equation [S1]:
    $$
    l = \text{median}\left(0, L_{max}, \left\lfloor \log_2 \frac{s(z)}{v_0} \right\rfloor \right)
    $$
    Where $s(z)$ is the variable step-size, $L_{max}$ is the maximum available mipmap level, and $v_0$ is the voxel size at the finest mipmap level [S1]. The implementation uses a derived formula:
    $$
    \text{mipmap\_l} = \log_2(S_0 + 2 \cdot \tan \Phi \cdot z) - \log_2 V
    $$
9.  **Texture Sampling:** Access the volume texture using the `tex3Dlod` intrinsic (or equivalent) with the computed mipmap level [S1].
10. **Compositing:** Accumulate color and opacity using front-to-back compositing [S1].
11. **Transfer Function:** Apply the transfer function and shading to get color $C_i$ and opacity $\alpha_i$ [S1].

**Pseudocode:**
The following pseudocode is derived from Appendix A of [S1]. Helper functions `clip_to_unit_box` and `mipmap_value` are assumed to be defined. `glstate.matrix.modelview` refers to the current model-view matrix.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Uniforms
Ew: float3          # Projector position
Vd: float           # Virtual observer distance
Vh: float           # Virtual observer height
log2V: float        # log2 voxel size
DL: float           # stepsize
S0: float           # screen pixel size
tanPhi: float       # angular spread factor
a_max: float        # termination threshold (typically 0.95 or 1.0) [G]

# Structures
struct ray_t:
    org: float3
    dir: float3
    len: float

struct fragment_out:
    color: float4

# Helper: Clip ray to unit box [S1]
# Returns float2 {tmin, tmax}
function clip_to_unit_box(origin: float3, direction: float3) -> float2:
    # Implementation details omitted in source, standard slab method assumed [G]
    pass

# Helper: Sample texture with LOD [S1]
function mipmap_value(uvw: float3, level: float) -> float4:
    # Uses tex3Dlod intrinsic [S1]
    pass

# Main Ray Casting Function
function ray_at_xy(x: float, y: float, mv_inv: float4x4) -> ray_t:
    result: ray_t
    
    # Compute ray passing through pixel in world coords [S1]
    ray_o: float4 = float4(x, y, 0, 0)
    ray_d: float4 = float4(-(Ew.x - x) / Ew.z, (Vh - y) / Vd, -1, 0)
    
    # Transform from world to texture coordinates [S1]
    tex_o: float3 = mul(mv_inv, ray_o).xyz
    tex_e: float3 = mul(mv_inv, ray_o + ray_d).xyz
    
    # Clip ray to volume box [S1]
    result.dir = normalize(tex_e - tex_o)
    tmin_tmax: float2 = clip_to_unit_box(tex_o, result.dir)
    
    result.org = tex_o + result.dir * tmin_tmax.x
    result.len = tmin_tmax.y - tmin_tmax.x
    
    return result

function fragment_shader(Qw: vertex_out) -> fragment_out:
    OUT: fragment_out
    
    # Compute ray in tex coords [S1]
    # IN.Qw.x and IN.Qw.y are the projected coordinates from vertex shader
    ray: ray_t = ray_at_xy(Qw.x, Qw.y, inverse_modelview_matrix)
    
    # If ray length is less than epsilon, discard fragment [S1]
    if ray.len < epsilon:
        discard
    
    else:
        # Ray composition initialization [S1]
        delta_dir: float3 = ray.dir * DL
        N: int = int(ray.len / DL)
        uvw: float3 = ray.org
        I: float3 = float3(0, 0, 0)
        a: float = 0.0
        
        for i from 0 to N-1:
            # Check early termination [S1]
            if a >= a_max:
                break
            
            # Find distance from screen [S1]
            # Transform uvw to world space to get z
            z: float = abs(dot(glstate.matrix.modelview[0][2], float4(uvw.xyz, 1.0)))
            
            # Compute mipmap level [S1]
            mipmap_l: float = log2(S0 + 2 * tanPhi * z) - log2V
            
            # Sample texture [S1]
            gggv: float4 = mipmap_value(uvw.xyz, mipmap_l)
            
            # Apply transfer function and shading [S1]
            # C_i and a_i are derived from gggv (RGB=gradient, A=voxel value)
            C_i: float3 = ... # Implementation details omitted in source [S1]
            a_i: float = ...  # Implementation details omitted in source [S1]
            
            # Accumulate color and opacity [S1]
            I = I + (1.0 - a) * C_i
            a = a + (1.0 - a) * a_i
            
            # Step along ray [S1]
            uvw += delta_dir
            
        OUT.color = float4(I, a)
    
    return OUT
```

**Edge Cases and Pitfalls:**
*   **Vertex Attribute Interpolation Error:** Standard vertex attribute interpolation does not produce correct per-pixel ray parameter values for MCOP. This is mitigated by using the proxy geometry and computing projections in the vertex shader [S1].
*   **Tile Misalignment Artifacts:** On the specific hardware (96 projectors updated in batches of 16), misalignment between tiles can become visible if objects are moved with a slow refresh rate [S1].
*   **Out of Bounds:** If the clipped ray length is null or less than epsilon, the fragment is discarded [S1].
*   **Performance:** At full resolution (1 ray/pixel, 1 sample/voxel), millions of rays require hundreds of steps, limiting interactive frame rates. The method suggests lowering sampling rate during interaction by reducing pixel count (2.5mm precision) and doubling the integration step-size [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of rays (pixels). The number of samples $N$ is determined by the ray length divided by the step size $DL$ [S1].
*   **Memory Complexity:** Dominated by the storage of the 3D volume texture (4-component) and the 1D LUT [S1].
*   **Measured Performance:** The method was demonstrated on a display with 96 projectors. Performance was limited by single GPU volume renderer performance at full resolution, necessitating reduced sampling rates for interactivity [S1].
*   **Strengths:** Provides a fully continuous perceived image for multiple viewers, unlike discrete multiview systems. Supports parallax, accommodation, and stereo cues [S1].
*   **Limitations:** Requires a specific light field display hardware setup. Tile misalignment artifacts can occur with slow refresh rates [S1].
*   **When to use:** Interactive direct volume rendering on multi-viewer light field displays where continuous viewing zones are required [S1].
*   **When not to use:** Standard single-viewer rendering or displays without light field capabilities [S1].

## Background
*   **Origin:** The method was introduced by Agus et al. in 2008 [S1].
*   **History:** It was developed to address the limitations of contemporary multiview technologies that forced users into fixed positions due to abrupt view-image changes [S1]. The approach builds on GPU-accelerated ray casting techniques and MCOP projection models [S1].
*   **Usage:** Used for interactive light field volume rendering, allowing users to manipulate objects with 6-DOF trackers and modify transfer functions while maintaining multi-user capability [S1].

## Sources
[S1] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
