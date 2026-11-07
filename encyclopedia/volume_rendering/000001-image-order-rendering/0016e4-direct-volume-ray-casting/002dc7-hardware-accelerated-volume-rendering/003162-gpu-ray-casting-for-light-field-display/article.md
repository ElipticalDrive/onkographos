# GPU Ray Casting for Light Field Display

Also called: Direct Volume Rendering on Light Field Display, Interactive Light Field Volume Rendering, MCOP Ray Casting with Adaptive Mipmap Level Selection, MCOP projection, Multi-viewer 3D Volume Rendering

## Overview
This method performs interactive direct volume rendering on an interactive light field display by computing rays from projector positions through screen pixels into the volume using Multi-Camera Optical Projection (MCOP) [S1]. It sets itself apart from standard GPU ray casting by using a coarsely tessellated proxy geometry to compute correct per-pixel ray parameters, avoiding interpolation errors inherent in standard vertex attribute interpolation, and employs depth-dependent adaptive mipmap level selection to match sampling resolution to the display's spatial constraints [S1]. Unlike traditional multiview technologies that enforce fixed viewing zones, this approach provides a fully continuous perceived image for multiple users [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Volume data $V$ stored in a 4-component 3D texture (RGB: unscaled precomputed gradient; A: voxel value) [S1].
    *   A 1D RGBA lookup table for the transfer function [S1].
    *   Uniform parameters: Projector position $E_w$ (float3), Virtual observer distance $V_d$ (float), Virtual observer height $V_h$ (float), $\log_2$ of voxel size $\log_2 V$ (float), Step size $DL$ (float), Screen pixel size $S_0$ (float), Angular spread factor $\tan \Phi$ (float) [S1].
    *   Model-view matrix $M_{mv}$ and its inverse $M_{mv}^{-1}$ [S1].
*   **Outputs:**
    *   Final color and opacity per fragment [S1].

**Data Structures:**
*   `ray_t`: Struct containing `org` (float3), `dir` (float3), and `len` (float) [S1].
*   `fragment_out`: Struct containing `color` (float4) [S1].

**Steps:**

1.  **Proxy Geometry Setup:** Render a coarsely tessellated proxy geometry (8x8 quads) representing a slightly enlarged bounding volume [S1].
2.  **Vertex Shader (MCOP Projection):** For each vertex of the proxy, compute the MCOP projection to determine the screen pixel position $S$ in the display reference frame [S1]. Pass normalized and unnormalized projected coordinates to the fragment stage [S1].
3.  **Fragment Shader - Ray Initialization:**
    *   Receive the unnormalized projected position $S$ (represented by screen coordinates $x, y$) [S1].
    *   Compute the ray direction $d$ in world coordinates using the MCOP projection formula [S1]:
        $$
        d = \left[ \frac{S_x - E_x}{E_z}, \frac{S_y - V_y}{V_z}, -1 \right]
        $$
        where $E = (E_x, E_y, E_z)$ is the projector position and $V = (V_x, V_y, V_z)$ defines the virtual viewer parameters (specifically $V_y=V_h, V_z=V_d$ based on uniform definitions) [S1].
    *   Transform the ray origin and a point along the direction from world space to texture (local volume) coordinates using the inverse model-view matrix [S1]:
        $$
        \text{tex\_o} = M_{mv}^{-1} \cdot \text{ray\_o}
        $$
        $$
        \text{tex\_e} = M_{mv}^{-1} \cdot (\text{ray\_o} + \text{ray\_d})
        $$
    *   Clip the line segment defined by $\text{tex\_o}$ and direction against the unit box $[0,1]^3$ to find the entry distance $t_{min}$ and exit distance $t_{max}$ [S1].
    *   Calculate the ray length $L = t_{max} - t_{min}$ [S1].
    *   If $L < \epsilon$ (epsilon), discard the fragment as it is out of bounds [S1].
    *   Set the ray origin in texture space: $\text{org} = \text{tex\_o} + \text{dir} \cdot t_{min}$ [S1].
4.  **Ray Marching Loop:**
    *   Initialize compositing variables: accumulated color $I = (0,0,0)$, accumulated opacity $a = 0.0$, current position $\text{uvw} = \text{org}$, step vector $\text{delta\_dir} = \text{dir} \cdot DL$, and total steps $N = \lfloor L / DL \rfloor$ [S1].
    *   Loop for $i$ from 0 to $N-1$:
        *   **Termination Check:** If $a \ge a_{max}$ (typically 1.0), break loop (Early Ray Termination) [S1].
        *   **Depth Calculation:** Compute the depth $z$ of the current sample $\text{uvw}$ from the screen (world-aligned z-coordinate) using a dot product with the model-view matrix [S1]:
            $$
            z = | \mathbf{m}_{mv}[2] \cdot \text{uvw} |
            $$
            where $\mathbf{m}_{mv}[2]$ is the third row of the model-view matrix [S1].
        *   **Mipmap Level Selection:** Compute the adaptive mipmap level $l$ to match the spatial resolution at depth $z$ [S1]:
            $$
            l = \text{median}\left(0, L_{max}, \left\lfloor \log_2 \left( \frac{s(z)}{v_0} \right) \right\rfloor \right)
            $$
            In the implementation, this is approximated/calculated as:
            $$
            l = \log_2(S_0 + 2 \cdot \tan \Phi \cdot z) - \log_2 V
            $$
            where $s(z)$ is the variable step-size, $v_0$ is the finest voxel size, and $L_{max}$ is the maximum mipmap level [S1].
        *   **Sampling:** Sample the volume texture at $\text{uvw}$ using the computed mipmap level $l$ via `tex3Dlod` (or equivalent) to get value $g_{gggv}$ [S1].
        *   **Classification/Shading:** Apply the transfer function (using the 1D LUT) and shading model (e.g., Phong, gradient-based) to derive color $C_i$ and opacity $\alpha_i$ from $g_{gggv}$ [S1].
        *   **Compositing:** Update accumulated color and opacity using front-to-back over operator [S1]:
            $$
            I = I + (1.0 - a) C_i
            $$
            $$
            a = a + (1.0 - a) \alpha_i
            $$
        *   **Step:** Advance position: $\text{uvw} = \text{uvw} + \text{delta\_dir}$ [S1].
5.  **Output:** Return the final accumulated color and opacity [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Uniforms
Ew: float3       # Projector position
Vd: float        # Virtual observer distance
Vh: float        # Virtual observer height
log2V: float     # log2 of voxel size
DL: float        # Step size
S0: float        # Screen pixel size
tanPhi: float    # Angular spread factor
mv_inv: float4x4 # Inverse model-view matrix
mv: float4x4     # Model-view matrix

# Helper: Clip ray to unit box [0,1]^3
def clip_to_unit_box(origin: float3, direction: float3) -> float2:
    # Returns tmin, tmax intersection distances with unit box
    # Implementation details omitted as per source structure, 
    # assuming standard slab method logic.
    pass

# Helper: Sample texture with mipmap level
def mipmap_value(position: float3, level: float) -> float4:
    # Performs 3D texture lookup with specified mipmap LOD
    pass

# Helper: Apply transfer function and shading
def classify_and_shade(value: float4) -> tuple[float3, float]:
    # value.rgb = gradient, value.a = voxel value
    # Returns (color, opacity)
    pass

struct ray_t:
    org: float3
    dir: float3
    len: float

def ray_at_xy(x: float, y: float, mv_inv: float4x4) -> ray_t:
    # Compute ray passing through pixel in world coords
    ray_o = float4(x, y, 0, 0)
    # MCOP Ray Direction Calculation
    # d = [ (Sx - Ex)/Ez , (Sy - Vy)/Vz , -1 ]
    # Note: Source uses (Vh-y)/Vd for Y component. 
    # Assuming V_y corresponds to Vh and V_z to Vd in the specific display frame.
    ray_d = float4(-(Ew.x - x) / Ew.z, (Vh - y) / Vd, -1, 0)
    
    # Transform from world to texture coordinates
    tex_o = mul(mv_inv, ray_o).xyz
    tex_e = mul(mv_inv, ray_o + ray_d).xyz
    
    # Clip ray to volume box
    dir_norm = normalize(tex_e - tex_o)
    tmin_tmax = clip_to_unit_box(tex_o, dir_norm)
    
    result = ray_t()
    result.dir = dir_norm
    result.org = tex_o + dir_norm * tmin_tmax.x
    result.len = tmin_tmax.y - tmin_tmax.x
    return result

def fragment_shader(x: float, y: float) -> float4:
    # Compute ray in tex coords
    ray = ray_at_xy(x, y, mv_inv)
    
    # If ray length is less than epsilon, discard fragment
    if ray.len < 1e-6:
        return float4(0, 0, 0, 0)
    
    # Ray composition initialization
    delta_dir = ray.dir * DL
    N = int(ray.len / DL)
    uvw = ray.org
    I = float3(0, 0, 0)
    a = 0.0
    a_max = 0.95 # Typical threshold, or 1.0
    
    for i in range(N):
        if a >= a_max:
            break
            
        # Find distance from screen (z coordinate in world space)
        # Using the third row of the model-view matrix
        z = abs(dot(mv[2], float4(uvw.xyz, 1.0)))
        
        # Compute mipmap level
        # l = log2(S0 + 2*tanPhi*z) - log2V
        mipmap_l = log2(S0 + 2 * tanPhi * z) - log2V
        
        # Clamp mipmap level to valid range [0, Lmax]
        # Lmax is not explicitly passed but implied by texture hierarchy
        # Implementation note: clamp to texture's max level
        mipmap_l = clamp(mipmap_l, 0.0, float(texture_max_level))
        
        # Sample texture at current position with computed mipmap level
        gggv = mipmap_value(uvw, mipmap_l)
        
        # Apply transfer function and shading
        C_i, alpha_i = classify_and_shade(gggv)
        
        # Accumulate color and opacity using front-to-back compositing
        I = I + (1.0 - a) * C_i
        a = a + (1.0 - a) * alpha_i
        
        # Step along ray
        uvw += delta_dir
        
    return float4(I, a)
```

**Edge Cases and Pitfalls:**
*   **Vertex Attribute Interpolation Error:** Standard fixed-function pipeline interpolation produces incorrect per-pixel ray parameters for MCOP because the projection cannot be recast into a traditional homogeneous matrix and long lines appear curved. This is mitigated by using the proxy geometry and computing projections in the vertex shader [S1].
*   **Tile Misalignment:** The 96 display projectors are updated in batches of 16. If objects are moved with a slow refresh rate, misalignment between tiles becomes visible [S1].
*   **Out of Bounds:** If the clipped ray length is null or less than epsilon, the fragment is discarded [S1].

## Operating characteristics
*   **Time Complexity:** The rendering cost is proportional to the number of pixels (projectors) multiplied by the number of samples per ray. The number of samples is determined by the ray length divided by the step size $DL$ [S1].
*   **Memory Complexity:** Requires storage for the 3D volume texture (4-component) and the 1D transfer function texture [S1].
*   **Performance:** At full resolution (1 ray/pixel, 1 sample/voxel), millions of rays are propagated, limiting interactive frame rates. Performance is improved by lowering the sampling rate (reducing pixel count to 2.5mm precision and doubling the integration step-size) during interaction [S1].
*   **Strengths:** Provides a fully continuous perceived image for multiple users, unlike discrete multiview systems. Correctly handles parallax and accommodation cues [S1].
*   **Limitations:** Performance is constrained by single GPU throughput. Batched projector updates can cause visible artifacts during fast motion [S1].

## Background
This method was introduced by Agus et al. in 2008 for rendering on an interactive light field display [S1]. It addresses the limitations of traditional multiview technologies that force users into fixed positions due to abrupt view changes when crossing discrete viewing zones [S1]. The approach leverages GPU acceleration to handle the high computational load of ray casting for multiple viewpoints simultaneously [S1].

## Sources
[S1] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
