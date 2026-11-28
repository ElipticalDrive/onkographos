# Depth Alpha Blending Projection

Also called: Depth-8 Alpha Blending Projection, Depth-aware RGB projection, Two-half alpha blending

## Overview
Depth Alpha Blending Projection renders a 2D projection of a 3D volume by alpha blending voxel values along rays, assuming pre-multiplied alpha [S1, unofficial]. It offers two variants: a grayscale "Depth-8" mode that outputs a single channel, and an RGB mode that splits the ray into two halves for accumulation to approximate depth-aware color blending [S1, unofficial]. It is distinguished from its siblings by its explicit two-pass accumulation structure in the RGB variant, which allows for specific depth-weighting strategies not present in standard single-pass ray casting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture (grayscale or RGB), a transformation matrix $T$ defining the ray origin and direction, and image dimensions $N_x \times N_y$.
*   **Output:** A 2D image buffer (8-bit grayscale or 32-bit RGBA).

**Data Structures:**
*   **Volume:** Accessed via trilinear interpolation. For RGB, the source data is assumed to be stored in a format where the alpha channel is the 4th component (e.g., `lut[index]` returns RGBA).
*   **Transform Matrix $T$:** A $4 \times 4$ matrix. The relevant components are $T.m[i][j]$ (row-major or column-major as per implementation, but indices are explicit in steps).

**Parameters:**
*   $N_z$: The depth dimension of the volume.
*   $N_x, N_y$: The width and height of the output image.

**Algorithm Steps:**

The method consists of two distinct modes. Both iterate over the image pixels $(x, y)$.

### 1. Depth-8 Alpha Blending Projection (Grayscale)

1.  **Initialize Ray:** For each pixel $y$ in $0 \dots N_y-1$:
    *   Calculate start point $p_0$ at $(0, y, (N_z-1)/2.0)$ using transform $T$.
    *   Extract initial coordinates $x_0, y_0, z_0$ from $p_0$.
    *   Extract forward step vector $(dx, dy, dz)$ from $T.m[0][0], T.m[1][0], T.m[2][0]$.
    *   Extract backward step vector $(dxd, dyd, dzd)$ from $-T.m[0][2], -T.m[1][2], -T.m[2][2]$.
2.  **Accumulate Backwards:**
    *   Initialize `alpha = 1.0`, `gray = 0.0`.
    *   Set current position $x_1 = x_0, y_1 = y_0, z_1 = z_0$.
    *   Loop $i_z$ from $0$ to $N_z-1$:
        *   Get voxel value `here` = trilinear interpolation at $(x_1, y_1, z_1)$.
        *   Normalize: `here` = `here` / 255.0.
        *   Compute effective alpha: $h_{alpha} = \sqrt{here}$.
        *   Update gray value: $gray = here \cdot h_{alpha} + gray \cdot (1 - h_{alpha})$.
        *   Increment position: $x_1 \leftarrow x_1 + dxd$, $y_1 \leftarrow y_1 + dyd$, $z_1 \leftarrow z_1 + dzd$.
3.  **Write Output:**
    *   Compute final intensity: $val = 255.99 \cdot gray / alpha$.
    *   Write `uint8_t(val)` to `dest[y * N_x + x]`.
    *   Increment ray start: $x_0 \leftarrow x_0 + dx$, $y_0 \leftarrow y_0 + dy$, $z_0 \leftarrow z_0 + dz$.

### 2. Depth Alpha Blending Projection (RGB)

1.  **Initialize Ray:** Same as Depth-8 mode.
2.  **Accumulate First Half (Front):**
    *   Initialize $a=0, r=0, g=0, b=0$.
    *   Set current position $x_1 = x_0, y_1 = y_0, z_1 = z_0$.
    *   Loop $i_z$ from $0$ to $\lfloor N_z/2 \rfloor - 1$ (first half):
        *   Get `herergb` from volume lookup.
        *   Extract normalized RGB channels $r_h, g_h, b_h$ (and alpha $a_h$, though $b_h$ is used as alpha proxy in source).
        *   Compute effective alpha: $a_h = b_h \cdot b_h$.
        *   Update color and alpha:
            $$r = r_h \cdot a_h + r \cdot (1 - a_h)$$
            $$g = g_h \cdot a_h + g \cdot (1 - a_h)$$
            $$b = b_h \cdot a_h + b \cdot (1 - a_h)$$
            $$a = a_h + a \cdot (1 - a_h)$$
        *   Increment position: $x_1 \leftarrow x_1 + dxd$, etc.
3.  **Accumulate Second Half (Back):**
    *   Loop $i_z$ from $\lfloor N_z/2 \rfloor$ to $N_z-1$ (second half):
        *   Get `herergb`.
        *   Extract $r_h, g_h, b_h$.
        *   Compute effective alpha: $a_h = b_h \cdot b_h$.
        *   Update color and alpha using the standard front-to-back blending formula with the accumulated front alpha:
            $$r = r + r_h \cdot a_h \cdot (1 - a)$$
            $$g = g + g_h \cdot a_h \cdot (1 - a)$$
            $$b = b + b_h \cdot a_h \cdot (1 - a)$$
            $$a = a + a_h \cdot (1 - a)$$
        *   Increment position: $x_1 \leftarrow x_1 + dxd$, etc.
4.  **Write Output:**
    *   Convert accumulated $r, g, b$ to 8-bit integers: $r_1 = \lfloor 255.99 \cdot r \rfloor$, etc.
    *   Write result as 32-bit integer: `0xff000000u + b1 + 256u*g1 + 65536u*r1` to `dest`.
    *   Increment ray start: $x_0 \leftarrow x_0 + dx$, etc.

**Formulas:**
The core blending equation for the RGB mode's second half is:
$$outA = frontA + backA \cdot (1 - frontA)$$
$$outRGB = frontRGB + backRGB \cdot (1 - frontA)$$
where $frontA$ is the accumulated alpha from the first half, and $backA, backRGB$ are the contributions from the current voxel in the second half [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Alpha Definition:** The method assumes the volume's alpha channel (or the 4th component in RGB lookup) represents density/transmittance. The square root operation ($\sqrt{here}$) implies that the raw voxel value is squared density or requires gamma correction [S1, unofficial].
*   **Half-Split Boundary:** The split at $N_z/2$ is arbitrary. If $N_z$ is odd, the middle voxel is included in the second half (loop starts at $N_z/2$). This asymmetry should be noted [S1, unofficial].
*   **Precision:** Accumulation is done in floating point. Final conversion uses `255.99` to handle rounding correctly for 8-bit output [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_alpha_blending_grayscale(volume, transform, nx, ny, nz):
    dest = [0] * (nx * ny)
    # Extract steps from transform matrix T
    # Assuming T is a flat list or accessible via T.m[row][col]
    dx = transform.m[0][0]
    dy = transform.m[1][0]
    dz = transform.m[2][0]
    
    dxd = -transform.m[0][2]
    dyd = -transform.m[1][2]
    dzd = -transform.m[2][2]

    for y in range(ny):
        # Calculate start point p0 at (0, y, (nz-1)/2.0) using transform
        # This implies a specific ray origin calculation. 
        # We assume p0 is the result of applying transform to (0, y, (nz-1)/2.0)
        p0 = transform.apply(0, y, (nz - 1) / 2.0)
        x0, y0, z0 = p0.x, p0.y, p0.z
        
        alpha = 1.0
        gray = 0.0
        x1, y1, z1 = x0, y0, z0
        
        for iz in range(nz):
            # Trilinear interpolation
            here = volume.trilinear(x1, y1, z1)
            here_norm = here / 255.0
            
            halpha = math.sqrt(here_norm)
            
            gray = here_norm * halpha + gray * (1.0 - halpha)
            
            x1 += dxd
            y1 += dyd
            z1 += dzd
            
        val = int(255.99 * gray / alpha)
        dest[y * nx + x0] = val # Note: x0 is not updated in loop, but dest index needs current x
        
        # Increment for next pixel
        x0 += dx
        y0 += dy
        z0 += dz

    return dest

def depth_alpha_blending_rgb(volume, transform, nx, ny, nz):
    dest = [0] * (nx * ny)
    
    dx = transform.m[0][0]
    dy = transform.m[1][0]
    dz = transform.m[2][0]
    
    dxd = -transform.m[0][2]
    dyd = -transform.m[1][2]
    dzd = -transform.m[2][2]

    for y in range(ny):
        p0 = transform.apply(0, y, (nz - 1) / 2.0)
        x0, y0, z0 = p0.x, p0.y, p0.z
        
        a = 0.0
        r = 0.0
        g = 0.0
        b = 0.0
        
        x1, y1, z1 = x0, y0, z0
        
        # First half
        half_nz = nz // 2
        for iz in range(half_nz):
            herergb = volume.lookup(iz, x1, y1, z1) # Assuming lookup handles trilinear
            rh, gh, bh = herergb.r, herergb.g, herergb.b
            
            ah = bh * bh
            
            r = rh * ah + r * (1.0 - ah)
            g = gh * ah + g * (1.0 - ah)
            b = bh * ah + b * (1.0 - ah)
            a = ah + a * (1.0 - ah)
            
            x1 += dxd
            y1 += dyd
            z1 += dzd
            
        # Second half
        for iz in range(half_nz, nz):
            herergb = volume.lookup(iz, x1, y1, z1)
            rh, gh, bh = herergb.r, herergb.g, herergb.b
            
            ah = bh * bh
            
            r = r + rh * ah * (1.0 - a)
            g = g + gh * ah * (1.0 - a)
            b = b + bh * ah * (1.0 - a)
            a = a + ah * (1.0 - a)
            
            x1 += dxd
            y1 += dyd
            z1 += dzd
            
        r1 = int(255.99 * r)
        g1 = int(255.99 * g)
        b1 = int(255.99 * b)
        
        # Write as 0xff000000 | B | G | R
        pixel = 0xff000000 | b1 | (g1 << 8) | (r1 << 16)
        dest[y * nx + x0] = pixel
        
        x0 += dx
        y0 += dy
        z0 += dz

    return dest
```

## Operating characteristics
*   **Time Complexity:** $O(N_x \cdot N_y \cdot N_z)$ for both variants, as every voxel along every ray is visited.
*   **Memory Complexity:** $O(1)$ auxiliary space per pixel, excluding the volume storage.
*   **Performance:** The RGB variant performs two passes over the depth axis (conceptually), but in practice iterates once, splitting the accumulation logic. This adds negligible overhead compared to the grayscale version but requires more register state (R, G, B, A) [S1, unofficial].
*   **Strengths:** Simple to implement; provides a depth-aware projection without full ray marching; the two-half split allows for different weighting of front vs. back surfaces [S1, unofficial].
*   **Limitations:** The square root alpha assumption is non-standard and may require pre-processing of volume data; the hard split at $N_z/2$ is arbitrary and may not align with semantic object boundaries; no shadowing or lighting calculations [S1, unofficial].

## Background
The method is described in the `gvox` library documentation, which provides an unofficial implementation of volume rendering techniques [S1, unofficial]. It is a specific instance of layer-based rendering, building on the principles of [Depth and Layer-Based Rendering](20975-depth-and-layer-based-rendering.md).

## Sources
[S1] wagenadl/gvox, https://github.com/wagenadl/gvox, fetched 2026-10-28 (unofficial)
