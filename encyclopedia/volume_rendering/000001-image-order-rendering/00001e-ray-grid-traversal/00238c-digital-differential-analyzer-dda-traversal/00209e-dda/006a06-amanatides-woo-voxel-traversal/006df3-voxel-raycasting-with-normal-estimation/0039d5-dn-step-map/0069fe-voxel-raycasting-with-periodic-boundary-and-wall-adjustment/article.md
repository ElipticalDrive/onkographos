# Voxel Raycasting with Periodic Boundary and Wall Adjustment

Also called: Periodic Voxel Raycasting, Wall-Adjusted Voxel Traversal

## Overview
This method renders a 3D voxel map by stepping through voxels along a ray, specifically designed to handle periodic boundary conditions (wrapping) and ensure that all six faces of a voxel cube are visible regardless of the viewing direction [S1, unofficial]. It builds on [DN_step_map](8596-dn-step-map.md) by adding coordinate wrapping and face-selection logic to the standard DDA traversal. Unlike [Voxel Raycast (Nonzero/Color Channel)](12412-voxel-raycast-nonzero-color-channel.md) which may assume infinite or bounded non-periodic space, this method treats the volume as a torus, and unlike [Voxel DDA Raymarching with Adaptive Edge Smoothing](14664-voxel-dda-raymarching-with-adaptive-edge-smoothing.md), it does not perform anti-aliasing but instead focuses on correct geometric visibility of voxel walls in a wrapped space.

## How it works
**Inputs:**
- `eye`: Camera position (float3).
- `forward`, `right`: Camera orientation vectors (float3). `right` is perpendicular to `forward`.
- `width`, `height`: Image dimensions (int).
- `maxdist`: Maximum ray distance (float).
- `voxel_size`: Dimensions of the voxel map (int3, e.g., 64x64x64) [S1, unofficial].
- `voxel_rgba`: 3D array of voxel colors (RGBA floats), indexed using strides `voxel_xstride`, `voxel_ystride`, `voxel_zstride` [S1, unofficial].

**Outputs:**
- `color`: Output image (PPM format, 8-bit RGB) and depth map (PGM format, 8-bit grayscale) [S1, unofficial].

**Data Structures:**
- `Float4`: Vector type with x, y, z, w components.
- `Float3`: Vector type with x, y, z components.
- `Int4`: Integer vector type with x, y, z, w components.
- `voxel_rgba`: Flat array or 3D array accessed via strides. Strides are computed as:
  - `voxel_xstride = 1`
  - `voxel_ystride = voxel_size.x`
  - `voxel_zstride = voxel_ystride * voxel_size.y` [S1, unofficial]

**Parameters:**
- `voxel_size`: Typically 64x64x64 in the reference implementation [S1, unofficial].
- `maxdist`: Typically 128.0f or 9.0f depending on scene scale [S1, unofficial].

**Steps:**

1. **Camera Setup:**
   Compute the `up` vector as perpendicular to `forward` and `right`, with length scaled by the height/width ratio:
   $$ \text{up} = \text{normalize}(\text{cross}(\text{forward}, \text{right})) \times |\text{right}| \times \frac{\text{height}}{\text{width}} $$
   Compute `rowstart` (top-left corner of the image plane):
   $$ \text{rowstart} = \text{eye} + \text{forward} - \text{right} + \text{up} $$
   Compute delta vectors for pixel traversal:
   $$ \text{dx} = \text{right} \times \frac{2.0}{\text{width}} $$
   $$ \text{dy} = \text{up} \times \frac{-2.0}{\text{height}} $$ [S1, unofficial]

2. **Pixel Loop:**
   For each pixel $(x, y)$:
   - Compute ray origin: $\text{pos} = \text{rowstart} + x \cdot \text{dx} + y \cdot \text{dy}$ [S1, unofficial]
   - Call `voxel_ray(eye, pos, maxdist)` to get color.
   - Clamp and quantize color components to 8-bit integers for PPM output:
     $$ r8 = \begin{cases} 0 & \text{if } c.x \le 0.0 \\ 255 & \text{if } c.x \ge 1.0 \\ \lfloor 0.5 + 255.0 \cdot c.x \rfloor & \text{otherwise} \end{cases} $$
     (Same for $g8, b8$) [S1, unofficial]
   - For depth PGM:
     $$ d8 = \begin{cases} 0 & \text{if } c.w \le 0.0 \\ 255 & \text{if } c.w \ge \text{maxdist} \\ \lfloor 0.5 + 255.0 \cdot \frac{c.w}{\text{maxdist}} \rfloor & \text{otherwise} \end{cases} $$ [S1, unofficial]

3. **Voxel Ray Tracing (`voxel_ray`):**
   - Initialize `color` to transparent (0,0,0,0).
   - Iterate through voxels along the ray using DDA traversal (as defined in [DN_step_map](8596-dn-step-map.md)).
   - For each voxel hit:
     - Compute integer position `posi` by flooring the current float position `pos`:
       $$ \text{posi} = \lfloor \text{pos} \rfloor $$
     - **Periodic Boundary Handling:** Wrap `posi` into positive voxel space:
       $$ \text{posi}.x = \text{posi}.x \pmod{\text{voxel\_size}.x} $$
       $$ \text{if } (\text{posi}.x < 0) \text{ posi}.x += \text{voxel\_size}.x $$
       (Same for $y, z$) [S1, unofficial]
     - **Wall Adjustment:** Adjust voxel cell coordinates to ensure all six faces are visible. If the intersection bit indicates a face was hit and the ray direction is negative, decrement the coordinate:
       - If intersection bit 1 is set and $\text{dir}.x < 0$, decrement $\text{posi}.x$.
       - If intersection bit 2 is set and $\text{dir}.y < 0$, decrement $\text{posi}.y$.
       - If intersection bit 4 is set and $\text{dir}.z < 0$, decrement $\text{posi}.z$.
       This ensures that when viewing towards negative axes, the outer face is seen before the inner face [S1, unofficial].
     - Retrieve voxel color `c` from `voxel_rgba` using `posi` and strides.
     - If $c.w \ge 1.0$ (opaque) or $0.0 < c.w < 1.0$ (translucent):
       $$ \text{color} = \text{color} + c \times (1.0 - \text{color}.w) $$
       If $c.w \ge 1.0$, break the loop [S1, unofficial].
   - Set `color.w` to the ray distance `pos.w` and return [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ppm(eye, forward, right, width, height, maxdist, voxel_size, voxel_rgba, voxel_xstride, voxel_ystride, voxel_zstride):
    # Camera setup
    up = normalize(cross(forward, right)) * length(right) * (height / width)
    rowstart = eye + forward - right + up
    dx = right * (2.0 / width)
    dy = up * (-2.0 / height)
    
    for y in range(height):
        pos = rowstart + y * dy
        for x in range(width):
            c = voxel_ray(eye, pos, maxdist, voxel_size, voxel_rgba, voxel_xstride, voxel_ystride, voxel_zstride)
            r8 = clamp_quantize(c.x)
            g8 = clamp_quantize(c.y)
            b8 = clamp_quantize(c.z)
            d8 = clamp_quantize_depth(c.w, maxdist)
            output_ppm(r8, g8, b8)
            output_pgm(d8)
            pos += dx

def clamp_quantize(val):
    if val <= 0.0: return 0
    if val >= 1.0: return 255
    return int(0.5 + 255.0 * val)

def clamp_quantize_depth(val, maxdist):
    if val <= 0.0: return 0
    if val >= maxdist: return 255
    return int(0.5 + 255.0 * (val / maxdist))

def voxel_ray(eye, pos, maxdist, voxel_size, voxel_rgba, voxel_xstride, voxel_ystride, voxel_zstride):
    color = Float4(0.0, 0.0, 0.0, 0.0)
    # DDA traversal setup (from DN_step_map)
    # ... (standard DDA initialization) ...
    
    while distance < maxdist:
        # Get current float position and direction from DDA
        current_pos = ... # From DDA state
        dir = ... # From DDA state
        intersection_bits = ... # From DDA state (bit 1: x, bit 2: y, bit 4: z)
        
        # Periodic boundary handling
        posi_x = floor(current_pos.x) % voxel_size.x
        if posi_x < 0: posi_x += voxel_size.x
        posi_y = floor(current_pos.y) % voxel_size.y
        if posi_y < 0: posi_y += voxel_size.y
        posi_z = floor(current_pos.z) % voxel_size.z
        if posi_z < 0: posi_z += voxel_size.z
        
        # Wall adjustment
        if (intersection_bits & 1) and dir.x < 0:
            posi_x -= 1
        if (intersection_bits & 2) and dir.y < 0:
            posi_y -= 1
        if (intersection_bits & 4) and dir.z < 0:
            posi_z -= 1
            
        # Retrieve voxel color
        index = posi_x * voxel_xstride + posi_y * voxel_ystride + posi_z * voxel_zstride
        c = voxel_rgba[index]
        
        # Blending
        if c.w > 0.0:
            color = color + c * (1.0 - color.w)
            if c.w >= 1.0:
                break
        
        # Advance DDA
        distance = ... # From DDA state
        # ... (advance DDA state) ...
        
    color.w = distance
    return color
```

**Edge Cases and Pitfalls:**
- **Floating-point modulo semantics:** Standard remainder operators in many languages return a result with the same sign as the dividend. This method requires a positive remainder for wrapping. If the result is negative, add the modulus [S1, unofficial].
- **Wall visibility:** Without wall adjustment, viewing towards negative axes shows inner faces before outer faces because voxel cells are treated as having only three faces by default. The adjustment ensures six faces per voxel are correctly visible [S1, unofficial].
- **DDA Calculation Errors:** Incorrect calculations for `xnext`, `xdelta`, `ynext`, `ydelta`, `znext`, `zdelta` can cause voxels to be displayed as intersecting planes instead of cube walls. These values must be non-negative, and `xnext.x`, `xdelta.x`, etc., should be integers [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of voxels intersected by the ray. In the worst case, $N$ is proportional to the volume dimensions.
- **Memory Complexity:** $O(V)$ for the voxel grid, where $V$ is the total number of voxels.
- **Performance:** The reference implementation renders a 512x512 image of a 64x64x64 voxel grid. Performance depends on the number of voxels intersected and the cost of modulo operations for periodic boundaries.
- **Strengths:** Correctly handles periodic (toroidal) voxel spaces; ensures all six faces of a voxel are visible regardless of viewing angle.
- **Limitations:** Modulo operations can be computationally expensive; no anti-aliasing or sub-pixel sampling is performed.
- **When to use:** When rendering voxel data that wraps around (e.g., infinite terrain, toroidal simulations) and correct face visibility is required.
- **When not to use:** When non-periodic boundaries are needed, or when high-quality anti-aliasing is required.

## Background
This method originates from a discussion on GameDev StackExchange regarding a voxel raycasting algorithm that was not working correctly, specifically failing to display voxel walls properly and lacking periodic boundary support [S1, unofficial]. The solution provided corrects the DDA traversal parameters and adds the periodic wrapping and wall adjustment logic. It is used in contexts where voxel data is inherently periodic, such as certain types of procedural generation or simulation environments.

## Sources
[S1] Voxel raycasting algorithm not working correctly, https://gamedev.stackexchange.com/questions/191245/voxel-raycasting-algorithm-not-working-correctly, fetched 2026-10-27 (unofficial)
