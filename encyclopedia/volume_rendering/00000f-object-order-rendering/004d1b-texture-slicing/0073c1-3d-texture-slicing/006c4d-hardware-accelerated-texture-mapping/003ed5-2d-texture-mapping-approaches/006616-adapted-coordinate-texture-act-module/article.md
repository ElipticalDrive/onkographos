# Adapted Coordinate Texture (ACT) Module

Also called: Adaptador de Coordenadas de Textura, Texturas Adaptador Coord. Textura, Texture Coordinate Adapter (ACT)

## Overview
The Adapted Coordinate Texture (ACT) module is a shader-based intermediary that transforms 3D texture coordinates provided by a Direct Volume Rendering (DVR) technique into 2D texture coordinates and color channel indices, enabling access to 3D volume data stored in a 2D texture array on hardware that only supports 2D textures [S1]. It allows volume data stored in a specific "mosaic" layout (consecutive slices distributed across multiple 2D textures and RGBA channels) to be accessed with trilinear interpolation [S1]. Unlike methods that require the DVR algorithm to be aware of the storage layout, the ACT module ensures the DVR technique remains independent of the storage method, handling all mapping logic internally [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D texture coordinates $(x, y, z)$ from the DVR technique, where $z$ represents the slice index.
*   **Output:** A grayscale value (voxel intensity) obtained via trilinear interpolation, which is subsequently converted to RGBA by the DVR's transfer function.

**Data Structures and Layout:**
The volume is stored in a vector of 2D textures $T$ of size $N_{tex}$ (typically 6 in the reference implementation) [S1].
*   **Mosaic Organization:** Slices are stored consecutively along color channels and texture units [S1].
*   **Tile Dimensions:** Each "mosaic" (or tile) within a texture channel stores $M_x \times M_y$ slices [S1].
*   **Channels:** Each texture supports $Maxch$ color channels (typically 4 for RGBA) [S1].
*   **Total Slices:** $S$ is the total number of slices in the volume model [S1].

**Parameters:**
*   $S$: Total number of slices (float, Uniform) [S1].
*   $M_x, M_y$: Dimensions of each tile in the texture storage (float, Uniform) [S1].
*   $Maxch$: Maximum number of color channels per texture (float, Constant, typically 4.0) [S1].
*   $T$: Array of 2D textures storing the volumetric model (sampler2D array, Uniform) [S1].

**Algorithm Steps:**
The ACT module performs two main tasks to simulate trilinear interpolation using bilinear hardware interpolation [S1].

1.  **Calculate Base Texture Unit and Channel:**
    Determine the primary texture unit $u_1$ and color channel $ch_1$ that contains the slice corresponding to the input $z$ coordinate.
    *   Calculate the total capacity of one texture unit in slices: $Stex = \min(M_x \times M_y, S)$ [S1].
    *   Calculate the linear slice index: $aux1 = \lfloor S \times p.z \rfloor$ [S1].
    *   Determine the texture unit index: $u_1 = \lfloor aux1 / (M_x \times M_y \times Maxch) \rfloor$ [S1]. *Note: The source code calculates this via intermediate `aux2`.*
    *   Determine the channel index: $ch_1 = aux1 \pmod{Maxch}$ [S1].

2.  **Calculate Local 2D Coordinates and Neighbors:**
    Calculate the 2D texture coordinates $(x_1, y_1)$ and $(x_2, y_2)$ for the two texels that bound the input $z$ in the z-direction, and the residual z-depth for interpolation [S1].
    *   Calculate the position within the current "block" of slices: $aux3 = \lfloor p.z \times Stex \rfloor$ [S1].
    *   Calculate the next slice index for boundary checking: $aux4 = \text{mod}(aux3 + 1.0, Stex)$ [S1].
    *   Calculate the 2D coordinates for the first texel ($texPos1$):
        $$texPos1.x = \frac{aux3}{M_x} + \frac{p.x}{M_x}$$
        $$texPos1.y = \frac{\lfloor aux3 / M_y \rfloor}{M_y} + \frac{p.y}{M_y}$$
    *   Calculate the 2D coordinates for the second texel ($texPos2$):
        $$texPos2.x = \text{fract}\left(\frac{aux4}{M_x}\right) + \frac{p.x}{M_x}$$
        $$texPos2.y = \frac{\lfloor aux4 / M_y \rfloor}{M_y} + \frac{p.y}{M_y}$$

3.  **Determine Second Texture Unit and Channel:**
    Determine the texture unit $u_2$ and channel $ch_2$ for the neighbor texel, handling potential overflow across texture boundaries [S1].
    *   Calculate the next channel index: $next = ch_1 + \text{step}(aux4, aux3)$ [S1]. The `step` function returns 1 if $aux4 > aux3$ (indicating a wrap-around or boundary crossing) and 0 otherwise.
    *   Calculate the second texture unit: $u_2 = u_1 + \text{step}(Maxch, next)$ [S1].
    *   Calculate the second channel: $ch_2 = next \pmod{Maxch}$ [S1].

4.  **Sample and Interpolate:**
    Sample the two texels and perform linear interpolation in the z-direction [S1].
    *   Sample the first value: $v_1 = T[u_1].texPos1[ch_1]$ [S1].
    *   Sample the second value: $v_2 = T[u_2].texPos2[ch_2]$ [S1].
    *   Calculate the interpolation weight based on the fractional part of the z-coordinate within the slice block: $w = (p.z \times Stex) - aux3$ [S1].
    *   Return the result: $result = \text{mix}(v_1, v_2, w)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: p (vec3: x, y, z), S (float), Mx (float), My (float)
# Constants: Maxch = 4.0
# Uniforms: T (sampler2D array of size 6)

def ACT(p):
    # Task 1: Calculate base texture unit u1 and channel ch1
    Mxy = Mx * My
    aux1 = floor(S * p.z)
    
    # Calculate which "block" of (Mxy * Maxch) slices we are in
    aux2 = floor(aux1 / Mxy)
    
    # Determine u1 and ch1 from aux2
    # u1 is the texture unit index, ch1 is the channel index within that unit
    u1 = int(floor(aux2 / Maxch))
    ch1 = int(mod(aux2, Maxch))
    
    # Task 2: Calculate 2D coordinates for interpolation neighbors
    Stex = min(Mx * My, S)
    
    # aux3 is the index of the slice within the current texture unit's capacity
    aux3 = floor(p.z * Stex)
    # aux4 is the index of the next slice, wrapped around Stex
    aux4 = mod(aux3 + 1.0, Stex)
    
    # Calculate 2D texture coordinates for the first neighbor (u1, ch1)
    texPos1_x = (aux3 / Mx) + (p.x / Mx)
    texPos1_y = (floor(aux3 / My) / My) + (p.y / My)
    texPos1 = vec2(texPos1_x, texPos1_y)
    
    # Calculate 2D texture coordinates for the second neighbor (u2, ch2)
    texPos2_x = fract(aux4 / Mx) + (p.x / Mx)
    texPos2_y = (floor(aux4 / My) / My) + (p.y / My)
    texPos2 = vec2(texPos2_x, texPos2_y)
    
    # Determine u2 and ch2 for the second neighbor
    # step(aux4, aux3) returns 1 if aux4 > aux3 (boundary crossed), else 0
    next_ch = ch1 + int(step(aux4, aux3))
    
    # If next_ch exceeds Maxch, we move to the next texture unit
    u2 = u1 + int(step(Maxch, next_ch))
    ch2 = int(mod(next_ch, Maxch))
    
    # Sample values
    # Note: texture2D(T[u], coord) returns vec4; we access channel [ch]
    v1 = texture2D(T[u1], texPos1)[ch1]
    v2 = texture2D(T[u2], texPos2)[ch2]
    
    # Calculate z-residual for interpolation weight
    # depth represents the fraction of z covered by one texture unit
    depth = min(1.0, Mxy / S)
    zini = aux2 * depth
    zres = (p.z - zini) / depth
    
    # Interpolate between v1 and v2 based on position within the slice block
    # The weight is the fractional part of (p.z * Stex)
    weight = (p.z * Stex) - aux3
    
    return mix(v1, v2, weight)
```

**Edge Cases and Pitfalls:**
*   **Overflow Control:** The calculation of $u_2$ and $ch_2$ must correctly handle cases where the next slice index crosses the boundary of the current texture unit's capacity (i.e., when $aux4 > aux3$) [S1]. The `step` function is used to avoid conditional branches, which are expensive on mobile GPUs [S1].
*   **Mobile GPU Performance:** Control flow operations (like `if` statements) are costly on mobile GPUs. The implementation minimizes control flow by using GLSL functions like `step` and `mix` [S1].
*   **DVR Independence:** The DVR technique must not be aware of the mosaic storage layout; it simply provides 3D coordinates. The ACT module encapsulates all mapping logic [S1].

## Operating characteristics
*   **Time Complexity:** The ACT module adds a constant number of arithmetic operations and texture lookups (two 2D texture samples) per voxel access. It does not increase the number of texture accesses compared to a native 3D texture lookup, nor does it add conditional control flow instructions [S1].
*   **Memory Complexity:** Memory usage is determined by the total number of slices $S$ and the texture capacity. The volume is split across $N_{tex}$ textures. For example, on an iPad2 with 8 texture units and max texture size $2048^2$, the maximum resolution achieved was $512^2 \times 384$ voxels [S1]. On desktop WebGL (GeForce 8800GT, 16 texture units, $8192^2$ max size), resolutions up to $512^2 \times 1024$ were tested [S1].
*   **Performance:** On mobile devices (iPad2), draw times were nearly constant across experiments sharing the same ACT module, suggesting that volume resolution does not significantly impact performance as long as it fits in memory [S1]. However, using the full ACT module (handling multiple mosaics) results in approximately double the draw time of a simplified ACT module (handling a single mosaic) for small volumes (e.g., $128^3$), due to the extra computations required for multi-mosaic support [S1].
*   **Strengths:** Enables high-resolution volume rendering on hardware with limited 3D texture support or limited texture units by efficiently packing data into 2D textures. It is hardware-agnostic regarding the DVR algorithm.
*   **Limitations:** The overhead of coordinate transformation is non-zero, though minimal. Performance is constrained by the maximum texture size and number of texture units of the target hardware.

## Background
The ACT module was introduced by J.R. Jiménez and J.M. Noguera in the paper "High Resolution Medical 3D Data Sets on Mobile Devices and WebGL" (2012) [S1]. The work was presented at the CEIG 2012 conference (Computer Graphics International) [S1]. The authors developed the module to enable direct volume rendering of high-resolution medical datasets (such as the CTHuman dataset, $512^2 \times 1877$ voxels) on platforms with limited 3D texture support, specifically mobile devices (iPad2) and WebGL browsers [S1]. The approach was chosen to study the scalability of texture-based rendering with increasing resolution and multiple textures, rather than to benchmark DVR performance against ray-tracing [S1].

## Sources
[S1] High Resolution Medical 3D Data Sets on Mobile Devices and WebGL, https://doi.org/10.2312/localchapterevents/ceig/ceig12/021-029, fetched 2026-10-26
