# Far-Terrain Grid Generation
Also called: FarGrid
## Overview
Far-Terrain Grid Generation produces a coarse, linear-space RGBA vertex grid and height lattice for rendering distant terrain features using analytic heightfield sampling [S1, unofficial]. It differs from sibling methods like [Terrain Generation via Noise](2412-terrain-generation-via-noise.md) by explicitly calculating local slope to drive surface color selection and hill shading, and by providing a dedicated map image generation path that includes underwater depth fading and contour banding [S1, unofficial].
## How it works
**Inputs:** A procedural terrain generator object `gen` providing `heightAt(x, z)`, `slopeAt(x, z)`, and `surface(gen, x, z, y, slope, c)`; grid dimensions $n$ (width/height in blocks); grid step size `step` (block units); sea level constant `SEA`; and a starting coordinate $(x_0, z_0)$ [S1, unofficial].
**Outputs:** A vertex position array `pos` (size $3 \times (n+2)^2$), a color array `col` (size $4 \times (n+2)^2$), and a height lattice `h` [S1, unofficial].

**Data Structures:**
- `h`: `(n+2) \times (n+2)` array storing height values. The extra margin allows gradient calculation at boundaries [S1, unofficial].
- `pos`: Flat array of floats. Vertex $k$ (row-major) at grid index $(i, j)$ has position `pos[k*3] = i * step`, `pos[k*3+1] = y + 0.5`, `pos[k*3+2] = j * step` [S1, unofficial].
- `col`: Flat array of uint8s. Vertex $k$ color is `col[k*4..k*4+3]` [S1, unofficial].

**Steps:**
1. **Generate Height Lattice:** Create `h` of size $(n+2) \times (n+2)$. For each grid point $(i, j)$ corresponding to world coordinates $(x, z)$, sample `h[i*(n+2)+j] = gen.heightAt(x, z)`. This includes a one-sample margin around the core $n \times n$ area [S1, unofficial].
2. **Calculate Local Slope:** For each vertex $k$ at $(i, j)$ with height $y = h[k]$:
   - Compute coarse gradient: $g_{coarse} = \frac{\max(|h[k+1] - h[k-1]|, |h[k+(n+2)] - h[k-(n+2)]|)}{2 \times step}$ [S1, unofficial].
   - Compute steepest per-block rise using `gen.slopeAt(x, z)` [S1, unofficial].
   - Set `slope = max(coarse_gradient, steepest_rise)` [S1, unofficial].
3. **Determine Surface Color:** Call `surface(gen, x, z, y, slope, c)` to obtain base color components $c[0..2]$ and block type [S1, unofficial].
4. **Apply Value Noise Factor:** Calculate variation factor $v$:
   $$v = 0.85 + 0.12 \times \text{fbm}(x \cdot 0.013, z \cdot 0.013, 4242, 2) - \delta$$
   where $\delta = 0.35 \times \text{smooth}(SEA - y, 0, 20)$ if the block type is sand and the vertex is underwater ($y < SEA$), otherwise $\delta = 0$ [S1, unofficial].
5. **Set Vertex Position and Color:**
   - Position: $pos[k \cdot 3] = i \cdot step$, $pos[k \cdot 3 + 1] = y + 0.5$, $pos[k \cdot 3 + 2] = j \cdot step$ [S1, unofficial].
   - Color: Convert linear color $c_{lin} = c \cdot v$ to sRGB.
   $$C_{srgb} = \text{round}\left(\min(1, \max(0, C_{lin}))^{2.2} \times 255\right)$$
   Set `col[k*4..k*4+3]` to $(C_{srgb, r}, C_{srgb, g}, C_{srgb, b}, 255)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def far_terrain_grid(gen, n, step, sea_level, x0, z0):
    width = n + 2
    h = [0.0] * (width * width)
    pos = [0.0] * (width * width * 3)
    col = [0] * (width * width * 4)

    # Step 1: Generate Height Lattice
    for i in range(width):
        for j in range(width):
            x = x0 + i * step
            z = z0 + j * step
            h[i * width + j] = gen.heightAt(x, z)

    # Step 2-5: Process Vertices
    for i in range(width):
        for j in range(width):
            k = i * width + j
            x = x0 + i * step
            z = z0 + j * step
            y = h[k]

            # Calculate Local Slope
            idx_left = k - 1
            idx_right = k + 1
            idx_up = k - width
            idx_down = k + width

            dx = abs(h[idx_right] - h[idx_left])
            dz = abs(h[idx_down] - h[idx_up])
            coarse_grad = max(dx, dz) / (2 * step)

            steep_rise = gen.slopeAt(x, z)
            slope = max(coarse_grad, steep_rise)

            # Determine Surface Color
            c, block_type = gen.surface(gen, x, z, y, slope, None)

            # Value Noise Factor
            fbm_val = fbm(x * 0.013, z * 0.013, 4242, 2)
            delta = 0.0
            if block_type == 'sand' and y < sea_level:
                delta = 0.35 * smooth(sea_level - y, 0, 20)

            v = 0.85 + 0.12 * fbm_val - delta

            # Set Position
            pos[k * 3] = i * step
            pos[k * 3 + 1] = y + 0.5
            pos[k * 3 + 2] = j * step

            # Set Color (Linear to sRGB)
            c_lin = [c[0] * v, c[1] * v, c[2] * v]
            c_srgb = [0, 0, 0]
            for ch in range(3):
                val = max(0.0, min(1.0, c_lin[ch]))
                c_srgb[ch] = round(val ** 2.2 * 255)

            col[k * 4] = c_srgb[0]
            col[k * 4 + 1] = c_srgb[1]
            col[k * 4 + 2] = c_srgb[2]
            col[k * 4 + 3] = 255

    return pos, col, h

def fbm(x, z, seed, octaves):
    # Standard 2D value noise FBM with specified octaves [G]
    val = 0.0
    amp = 1.0
    freq = 1.0
    for _ in range(octaves):
        val += amp * value_noise(x * freq, z * freq, seed)
        amp *= 0.5
        freq *= 2.0
    return val

def value_noise(x, z, seed):
    # Simple integer grid hash noise [G]
    ix = int(x) & 0xFF
    iz = int(z) & 0xFF
    h = (ix * 374761393 + iz * 668265263 + seed) & 0x7FFFFFFF
    return (h % 10000) / 10000.0

def smooth(a, b, c):
    # Linear interpolation clamped to [0, 1] [G]
    if c == b: return 0.0
    t = max(0.0, min(1.0, (a - b) / (c - b)))
    return t
```

**Parameters:**
- `n`: Grid resolution (core size). Typical values depend on view distance.
- `step`: Block size in world units.
- `SEA`: Sea level height.
- Noise seed: `4242` for FBM.
- Noise scale: `0.013`.
- Noise octaves: `2`.

**Edge Cases:**
- Boundary vertices: The lattice includes a margin, so neighbors for gradient calculation exist [S1, unofficial].
- Underwater sand: The delta term reduces brightness for underwater sand blocks [S1, unofficial].
- Precision: Float32 is used for heights; sRGB conversion uses rounding [S1, unofficial].

## Subtopics

### Map Image Generation
Generates a 2D map image (sRGBA) from the analytic heightfield, including sea depth, beaches, grass, forest, rock, snow, and hill shading [S1, unofficial]. It samples a lattice of size $(w+2) \times (hgt+2)$ and calculates gradients $g_x = (h[k+1]-h[k-1])/(2 \cdot step)$ and $g_z = (h[k+P]-h[k-P])/(2 \cdot step)$ for each pixel [S1, unofficial]. If underwater ($y < SEA - 0.5$), it calculates depth factor $d = smooth(SEA - y, 0, 22)$ and sets base color $r=70-d \cdot 46, g=150-d \cdot 82, b=170-d \cdot 48$; if shallow ($SEA-y < 2.5$), it adds $r+=30, g+=30, b+=20$ [S1, unofficial]. For land, it calls `surface` for base color $c$, calculates hill shading $S = \text{clamp}(0.55, 1.3, 1 - 0.35(g_x + g_z) \min(1, \frac{step}{2} + 0.5))$, and applies a contour band factor of $0.88$ if height changes by 32 blocks between neighbors, else $1$ [S1, unofficial]. Final color is $r = c[0] \cdot 255 \cdot S \cdot band$, etc. [S1, unofficial].

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance with hardware, resolution and scene: Not given in the sources.
Scaling: Not given in the sources.
Strengths and limitations: Not given in the sources.
When to use it and when not: Not given in the sources.

## Background
Part of the procedural terrain generation pipeline in the `2.5d-experiment` project [S1, unofficial].

## Sources
[S1] 12.8B destructible voxels rendered in the browser with Rust WASM custom engine, https://github.com/cktang88/2.5d-experiment, fetched 2026-10-26 (unofficial)
