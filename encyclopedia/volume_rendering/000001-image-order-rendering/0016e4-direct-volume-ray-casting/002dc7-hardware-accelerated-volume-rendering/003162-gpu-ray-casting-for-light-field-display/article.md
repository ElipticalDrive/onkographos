# GPU Ray Casting for Light Field Display

Also called: Direct Volume Rendering on Light Field Display, Interactive Light Field Volume Rendering, MCOP Ray Casting with Adaptive Mipmap Level Selection, MCOP projection, Multi-viewer 3D Volume Rendering

## Overview
This method performs direct volume rendering on an interactive light field display by computing rays from projector positions through screen pixels into the volume using Multi-Camera Optical Projection (MCOP). It solves the problem of incorrect per-pixel ray parameter values caused by standard vertex attribute interpolation in non-linear light field projections. It sets itself apart from standard GPU ray casting by explicitly handling the geometric distortion of light field displays via a proxy geometry and MCOP projection, and from other light field methods by using depth-dependent mipmap selection to maintain spatial resolution consistency across the continuous viewing zone.

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D volume data stored in a 4-component texture (RGB: precomputed gradient, A: voxel value); a 1D RGBA lookup table; camera parameters (projector position $E_w$, virtual observer distance $V_d$, virtual observer height $V_h$); display parameters (screen pixel size $S_0$, angular spread $\tan \Phi$); rendering parameters (step size $DL$, max opacity $a_{max}$, max mipmap level $L_{max}$, log2 voxel size $\log_2 V$).
*   **Outputs:** A 2D image where each pixel corresponds to a projector view, composited with color $C'$ and opacity $\alpha'$.

**Data Structures:**
*   `ray_t`: Struct containing `org` (float3, ray origin in texture space), `dir` (float3, normalized direction), `len` (float, integration length).
*   `fragment_out`: Struct containing `color` (float4, RGBA output).
*   **Volume Texture:** Single 4-component texture. RGB channels store unscaled precomputed gradients; Alpha channel stores voxel values.
*   **Lookup Table:** 1D RGBA texture for interactive transfer function modulation.
*   **Proxy Geometry:** Coarsely tessellated bounding volume (8x8 quads), slightly enlarged to cover the display frustum.

**Steps:**

1.  **Proxy Geometry Setup:** Render a proxy geometry consisting of 8x8 quads representing a slightly enlarged bounding volume of the scene. This geometry is used to compute correct ray parameters, avoiding interpolation errors inherent in standard fixed-function pipelines for MCOP projections [S1].

2.  **Vertex Shader (MCOP Projection):** For each vertex of the proxy geometry, compute the projected screen position $S$ using MCOP projection equations. Pass both normalized and unnormalized projected coordinates to the fragment shader [S1].
    *   The direction $d$ of the ray passing through screen point $S$ is computed using emitter $E$ and virtual viewer parameters $V$:
        $$
        d = \left[ \frac{S_x - E_x}{E_z}, \frac{S_y - V_y}{V_z}, -1 \right]
        $$
        where $E = (E_x, E_y, E_z)$ is the projector position and $V$ defines the virtual viewer parameters [S1].

3.  **Fragment Shader (Ray Initialization):**
    *   Receive the unnormalized projected position $S$ from the vertex stage. $S$ represents the screen pixel’s position in the display reference frame [S1].
    *   Compute the ray origin and direction in world coordinates. Assuming the screen is at $z=0$ and the projector is at $E_w$:
        $$
        \text{ray\_o} = (x, y, 0, 0)
        $$
        $$
        \text{ray\_d} = \left( -\frac{E_{w,x} - x}{E_{w,z}}, \frac{V_h - y}{V_d}, -1, 0 \right)
        $$
        where $x, y$ are screen coordinates, $E_w$ is the projector position, $V_d$ is virtual observer distance, and $V_h$ is virtual observer height [S1].
    *   Transform the ray origin and end point from world to texture coordinates using the inverse model-view matrix ($mv_{inv}$):
        $$
        \text{tex\_o} = mv_{inv} \cdot \text{ray\_o}.xyz
        $$
        $$
        \text{tex\_e} = mv_{inv} \cdot (\text{ray\_o} + \text{ray\_d}).xyz
        $$
    *   Normalize the direction and clip the ray against the unit box (texture space bounds $[0,1]^3$) to find entry ($t_{min}$) and exit ($t_{max}$) distances:
        $$
        \text{result.dir} = \text{normalize}(\text{tex\_e} - \text{tex\_o})
        $$
        $$
        [t_{min}, t_{max}] = \text{clip\_to\_unit\_box}(\text{tex\_o}, \text{result.dir})
        $$
    *   Compute final ray parameters:
        $$
        \text{result.org} = \text{tex\_o} + \text{result.dir} \cdot t_{min}
        $$
        $$
        \text{result.len} = t_{max} - t_{min}
        $$
    *   If $\text{result.len} < \epsilon$ (epsilon), discard the fragment as it is out of bounds [S1].

4.  **Ray Marching Loop:**
    *   Initialize compositing variables: accumulated color $I = (0,0,0)$, accumulated opacity $a = 0.0$, current texture coordinate $uvw = \text{result.org}$, step vector $\text{delta\_dir} = \text{result.dir} \cdot DL$, and total steps $N = \lfloor \text{result.len} / DL \rfloor$ [S1].
    *   Iterate for $i$ from 0 to $N-1$:
        *   **Mipmap Level Selection:** Compute the depth $z$ of the current sample in world space (screen-aligned) using a dot product with the model-view matrix's third row (z-axis):
            $$
            z = | \text{dot}(\text{glstate.matrix.modelview}[0][2], (\text{uvw.xyz}, 1.0)) |
            $$
        *   Calculate the mipmap level $l$ based on depth $z$, screen pixel size $S_0$, angular spread $\tan \Phi$, and voxel size $\log_2 V$:
            $$
            l = \log_2(S_0 + 2 \cdot \tan \Phi \cdot z) - \log_2 V
            $$
            *Note: The source provides this specific implementation formula. The theoretical description mentions $l = \text{median}(0, L_{max}, \lfloor \log_2(s(z)/v_0) \rfloor)$, where $s(z)$ is a variable step size. The code implementation uses the log2 formula directly [S1].*
        *   **Sampling:** Sample the volume texture at $uvw$ using the computed mipmap level $l$ via `tex3Dlod` (or equivalent intrinsic). Let the result be $gggv$.
        *   **Transfer Function & Shading:** Apply the transfer function (using the 1D LUT) and shading model to extract color $C_i$ and opacity $\alpha_i$ from $gggv$. (Implementation details omitted in source code, marked as `...` [S1]).
        *   **Compositing:** Update accumulated color and opacity using front-to-back compositing:
            $$
            I = I + (1.0 - a) \cdot C_i
            $$
            $$
            a = a + (1.0 - a) \cdot \alpha_i
            $$
        *   **Step:** Advance along the ray:
            $$
            uvw = uvw + \text{delta\_dir}
            $$
        *   **Termination:** If $a \ge a_{max}$, break the loop (Early Ray Termination) [S1].

5.  **Output:** Write the final accumulated color $I$ and opacity $a$ to the framebuffer [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Uniforms
Ew: float3          # Projector position
Vd: float           # Virtual observer distance
Vh: float           # Virtual observer height
log2V: float        # Log2 of voxel size
DL: float           # Step size
S0: float           # Screen pixel size
tanPhi: float       # Angular spread factor
a_max: float        # Max opacity threshold
mv_inv: float4x4    # Inverse model-view matrix
mv_matrix: float4x4 # Model-view matrix (for z-depth calculation)
volume_tex: sampler3D # Volume texture (RGB: gradient, A: value)
lut_tex: sampler1D  # Lookup table texture

struct Ray:
    org: float3
    dir: float3
    len: float

def clip_to_unit_box(origin: float3, direction: float3) -> tuple[float, float]:
    """
    Clips a ray against the unit box [0,1]^3.
    Returns (t_min, t_max) of intersection.
    If no intersection, returns (inf, -inf) or similar invalid range.
    Implementation details omitted for brevity; standard slab method.
    """
    pass

def ray_at_xy(x: float, y: float) -> Ray:
    # Compute ray passing through pixel in world coords
    ray_o = float4(x, y, 0, 0)
    ray_d = float4(-(Ew.x - x) / Ew.z, (Vh - y) / Vd, -1, 0)
    
    # Transform from world to texture coordinates
    tex_o = mul(mv_inv, ray_o).xyz
    tex_e = mul(mv_inv, ray_o + ray_d).xyz
    
    # Clip ray to volume box
    dir = normalize(tex_e - tex_o)
    tmin_tmax = clip_to_unit_box(tex_o, dir)
    
    # Check validity
    if tmin_tmax.y <= tmin_tmax.x:
        return Ray(float3(0,0,0), float3(0,0,0), 0)
        
    org = tex_o + dir * tmin_tmax.x
    length = tmin_tmax.y - tmin_tmax.x
    return Ray(org, dir, length)

def fragment_shader(screen_x: float, screen_y: float) -> float4:
    # Compute ray in tex coords
    ray = ray_at_xy(screen_x, screen_y)
    
    # If ray length is less than epsilon, discard fragment
    if ray.len < 1e-6:
        return float4(0, 0, 0, 0)
        
    # Initialize compositing variables
    delta_dir = ray.dir * DL
    N = int(ray.len / DL)
    uvw = ray.org
    I = float3(0, 0, 0)
    a = 0.0
    
    for i in range(N):
        # Find distance from screen (z coordinate in world space)
        # Using the third row of the model-view matrix
        z = abs(dot(mv_matrix[2], float4(uvw.xyz, 1.0)))
        
        # Compute mipmap level
        # Note: Source uses log2(S0 + 2*tanPhi*z) - log2V
        mipmap_l = log2(S0 + 2 * tanPhi * z) - log2V
        
        # Clamp mipmap level to valid range [0, L_max]
        # L_max is not explicitly passed as uniform in code snippet but implied by texture setup
        # Assuming L_max is available or texture handles clamping
        mipmap_l = clamp(mipmap_l, 0.0, float(L_max))
        
        # Sample texture at current position with computed mipmap level
        # gggv contains gradient (RGB) and value (A)
        gggv = sample_3d_lod(volume_tex, uvw, mipmap_l)
        
        # Apply transfer function and shading to get C_i and a_i
        # Implementation details omitted in source
        C_i = apply_transfer_function_and_shading(gggv)
        alpha_i = C_i.a # Or extracted from gggv.a depending on TF implementation
        
        # Accumulate color and opacity using front-to-back compositing
        I = I + (1.0 - a) * C_i.rgb
        a = a + (1.0 - a) * alpha_i
        
        # Step along ray
        uvw += delta_dir
        
        # Early Ray Termination
        if a >= a_max:
            break
            
    return float4(I, a)
```

**Edge Cases and Pitfalls:**
*   **Vertex Interpolation Error:** Standard vertex attribute interpolation fails for MCOP projections because the projection cannot be recast into a traditional homogeneous matrix, and long straight lines appear curved. Using a coarse proxy geometry (8x8 quads) and computing projections in the vertex shader mitigates this [S1].
*   **Tile Misalignment:** The display projectors are updated in batches of 16. If objects move with a slow refresh rate, misalignment between tiles becomes visible [S1].
*   **Out of Bounds:** If the clipped ray length is null or less than epsilon, the fragment is discarded [S1].
*   **Precision:** Mipmap level calculation relies on accurate depth estimation. The use of a single dot product for z-coordinate transformation is an optimization but assumes specific matrix layouts [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of samples per ray and $P$ is the number of pixels (projectors). $N$ depends on ray length and step size $DL$.
*   **Memory Complexity:** $O(V)$ for the volume texture, where $V$ is the number of voxels, plus $O(L)$ for the mipmap pyramid.
*   **Performance:** The method requires millions of rays to be propagated for hundreds of steps at full resolution (1 ray/pixel, 1 sample/voxel), which limits interactive frame rates. Performance is improved by lowering the sampling rate (reducing pixel count to 2.5mm precision and doubling step size) during interaction [S1].
*   **Strengths:** Provides a fully continuous perceived image for multiple viewers, unlike discrete multiview technologies. Correctly handles parallax and accommodation cues.
*   **Limitations:** Single GPU performance bottleneck at full resolution. Tile misalignment artifacts can occur with slow refresh rates and moving objects.

## Background
This method was introduced by Agus et al. in 2008 for rendering on an interactive light field display. It addresses the limitations of traditional multiview technologies that force users into fixed positions due to abrupt view-image changes. The MCOP approach allows for continuous viewing zones. The implementation uses a single GPU to accelerate the ray casting, leveraging mipmap-based sampling for adaptive resolution.

## Sources

[S1] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
