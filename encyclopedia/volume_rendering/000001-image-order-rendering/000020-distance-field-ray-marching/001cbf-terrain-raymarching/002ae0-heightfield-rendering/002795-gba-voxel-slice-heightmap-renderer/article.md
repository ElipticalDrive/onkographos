# GBA Voxel Slice Heightmap Renderer

Also called: Fixed-Point Voxel Slice Renderer, Precomputed Coordinate Voxel Renderer

## Overview
A software rendering technique for 3D voxel terrain on constrained hardware that iterates over discrete depth/height slices, maps screen columns to world coordinates using precomputed lookup tables and fixed-point math, samples a heightmap, and draws vertical lines to a frame buffer while maintaining a per-column height buffer for occlusion [S1, unofficial]. It builds on [Heightfield Rendering](1634-heightfield-rendering.md) by explicitly managing discrete depth slices with precomputed coordinate offsets (`matrizx`) to optimize the transformation loop for the Game Boy Advance's ARM7TDMI CPU, and differs from [Vertical Buffer Terrain Rendering](3924-vertical-buffer-terrain-rendering.md) by using a unified slice-based iteration structure rather than separate vertical buffer management [S1, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**:
  - Camera position: `posx`, `posy`, `posz` (fixed-point integers) [S1, unofficial].
  - Camera rotation state: `posr` (integer 0–11) [S1, unofficial].
  - Heightmap data: `rawprof` (heights) and `rawcolor` (colors) for a 1024x1024 world map [S1, unofficial].
  - Precomputed lookup tables: `matrizx` (x-offsets), `tempgfxp` (vertical line offsets) [S1, unofficial].
  - Rendering parameters: `minstep`, `maxstep`, `INC_STEP` [S1, unofficial].
- **Outputs**:
  - Frame buffer (VRAM) with terrain pixels drawn [S1, unofficial].
  - Per-column height buffer `alturas` updated for occlusion [S1, unofficial].

### Data Structures
- **`alturas[MODE5_WIDTH]`**: Array of `u8`. Stores the current drawn height per screen column. Size: 240 bytes. Initialized to 0 at the start of each frame [S1, unofficial].
- **`matrizx[numalturas * MODE5_WIDTH]`**: Array of `u16`. Precomputed lookup table storing x-offsets per column per depth step. Size: `numalturas * 240 * 2` bytes [S1, unofficial].
- **`punterosalturas[numalturas]`**: Array of `u8*` pointers. Each points to a contiguous block of heightmap data offset by `((maxstep-minstep+1)<<8)` [S1, unofficial].
- **`rawprof[1024 * 1024]`**: 1D array indexed by `x + (y << 10)`. Stores profile/height data for the 1024x1024 map [S1, unofficial].
- **`rawcolor[1024 * 1024]`**: 1D array indexed by `x + (y << 10)`. Stores color data for the 1024x1024 map [S1, unofficial].
- **`tempgfxp[MODE5_HEIGHT+1]`**: Array of `int`. Precomputed as `tempgfxp[i] = MODE5_WIDTH * (MODE5_HEIGHT - i)`. Used as byte offsets for vertical line drawing. Size: 161 * 4 bytes [S1, unofficial].

### Parameters
- `MODE5_WIDTH`: Screen width in pixels, typically 240 [S1, unofficial].
- `MODE5_HEIGHT`: Screen height in pixels, typically 160 [S1, unofficial].
- `dim_mapa`: World map dimension in pixels, defined as 1024 [S1, unofficial].
- `dim_mapa_log2`: Logarithm base 2 of map dimension, defined as 10 [S1, unofficial].
- `INC_STEP`: Depth slice iteration increment, computed as `((step>>6)+2)` [S1, unofficial].
- `posx`, `posy`, `posz`: Camera/position offsets. `posz` selects the heightmap slice index [S1, unofficial].
- `posr`: Rotation state (0–11), determining the coordinate transformation matrix [S1, unofficial].

### Numbered Steps
1. **Initialize Height Buffer**: Set `alturas[i] = 0` for all `i` in `[0, MODE5_WIDTH - 1]` [S1, unofficial].
2. **Iterate Depth Slices**: Loop `step` from `minstep` to `maxstep` with increment `INC_STEP` [S1, unofficial].
3. **Update Slice Index**: Compute `tmpalturas = step - minstep` [S1, unofficial].
4. **Update Background Registers**: Call `mueve_pantalla_step(tmpalturas)` to adjust background display registers [S1, unofficial].
5. **Load X-Offset Pointer**: Set `punterox = matrizx + MODE5_WIDTH * tmpalturas` [S1, unofficial].
6. **Scale Slice Index**: Compute `tmpalturas_scaled = tmpalturas << 8` (multiply by 256) [S1, unofficial].
7. **Iterate Screen Columns**: Loop `i` from 0 to `MODE5_WIDTH - 1` [S1, unofficial].
8. **Compute Intermediate Coordinates**:
   - `x = posx + punterox[i]` [S1, unofficial].
   - `y = posy + step` [S1, unofficial].
9. **Compute World Coordinates**: Apply rotation-specific transformation to `x` and `y` to get `xf` and `yf`, then mask with `(dim_mapa - 1)` to wrap within map bounds [S1, unofficial].
10. **Sample Heightmap**: Compute index `idx = tmpalturas_scaled + getprof(xf, yf)`. Read `alt = punterosalturas[posz][idx]` [S1, unofficial].
11. **Check Occlusion**: If `alt > alturas[i]`:
    - Draw vertical line from `alturas[i]` to `alt` with color `getcolor(xf, yf)` [S1, unofficial].
    - Update `alturas[i] = alt` [S1, unofficial].

### Formulas
- **Column Offset Lookup**: $x = posx + punterox[i]$ [S1, unofficial].
- **Slice Coordinate**: $y = posy + step$ [S1, unofficial].
- **World Coordinates**: Vary by `posr` (see rotation cases below) [S1, unofficial].
- **Heightmap Index**: $idx = tmpalturas\_scaled + getprof(xf, yf)$ [S1, unofficial].
  - $getprof(x, y) = rawprof[x + (y \ll dim\_mapa\_log2)]$ [S1, unofficial].
- **Color Lookup**: $getcolor(x, y) = rawcolor[x + (y \ll dim\_mapa\_log2)]$ [S1, unofficial].
- **Vertical Line Offset**: $offset = tempgfxp[alt]$ [S1, unofficial].

### Rotation Cases for World Coordinates
The transformation from screen-space intermediate coordinates $(x, y)$ to world coordinates $(xf, yf)$ depends on `posr`. All results are masked with `(dim_mapa - 1)` (i.e., `& 1023`). Note that integer division truncates toward zero [S1, unofficial].
- **Case 0, 3, 6, 9**:
  - $xf = (-y + x/2) \& (dim\_mapa - 1)$
  - $yf = (y/2 + x) \& (dim\_mapa - 1)$
- **Case 1, 4, 7, 10**:
  - $xf = -y \& (dim\_mapa - 1)$
  - $yf = x \& (dim\_mapa - 1)$
- **Case 2, 5, 8, 11**:
  - $xf = (-y/2 - x) \& (dim\_mapa - 1)$
  - $yf = (-y + x/2) \& (dim\_mapa - 1)$
- **Alternative mappings observed in source code**:
  - $xf = -x \& (dim\_mapa - 1)$, $yf = -y \& (dim\_mapa - 1)$
  - $xf = y \& (dim\_mapa - 1)$, $yf = -x \& (dim\_mapa - 1)$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MODE5_WIDTH = 240
MODE5_HEIGHT = 160
dim_mapa = 1024
dim_mapa_log2 = 10
dim_mapa_minus_1 = 1023

# Global/State Variables
posx = 0
posy = 0
posz = 0
posr = 0
minstep = 0
maxstep = 100
INC_STEP = 2  # Computed as ((step>>6)+2) in source

# Precomputed Data
# matrizx: u16 array of size numalturas * MODE5_WIDTH
# punterosalturas: u8* array of size numalturas
# rawprof: u8 array of size 1024*1024
# rawcolor: u8 array of size 1024*1024
# tempgfxp: int array of size MODE5_HEIGHT+1, precomputed as MODE5_WIDTH * (MODE5_HEIGHT - i)

def getprof(x, y):
    return rawprof[x + (y << dim_mapa_log2)]

def getcolor(x, y):
    return rawcolor[x + (y << dim_mapa_log2)]

def DrawVerticalLine(col, top, bottom, color):
    # Not given in the sources. Uses tempgfxp for byte offsets.
    pass

def mueve_pantalla_step(tmpalturas):
    # Not given in the sources. Updates background display registers.
    pass

def render_frame():
    # Initialize height buffer
    alturas = [0] * MODE5_WIDTH
    
    # Iterate depth slices
    step = minstep
    while step < maxstep:
        tmpalturas = step - minstep
        mueve_pantalla_step(tmpalturas)
        
        # Load x-offset pointer
        punterox = matrizx[tmpalturas * MODE5_WIDTH : (tmpalturas + 1) * MODE5_WIDTH]
        
        # Scale slice index
        tmpalturas_scaled = tmpalturas << 8
        
        # Iterate screen columns
        for i in range(MODE5_WIDTH):
            x = posx + punterox[i]
            y = posy + step
            
            # Compute world coordinates based on posr
            if posr in [0, 3, 6, 9]:
                xf = (-y + x // 2) & dim_mapa_minus_1
                yf = (y // 2 + x) & dim_mapa_minus_1
            elif posr in [1, 4, 7, 10]:
                xf = -y & dim_mapa_minus_1
                yf = x & dim_mapa_minus_1
            elif posr in [2, 5, 8, 11]:
                xf = (-y // 2 - x) & dim_mapa_minus_1
                yf = (-y + x // 2) & dim_mapa_minus_1
            else:
                # Fallback or other cases
                xf = -x & dim_mapa_minus_1
                yf = -y & dim_mapa_minus_1
            
            # Sample heightmap
            idx = tmpalturas_scaled + getprof(xf, yf)
            alt = punterosalturas[posz][idx]
            
            # Check occlusion and draw
            if alt > alturas[i]:
                DrawVerticalLine(i, alt, alturas[i], getcolor(xf, yf))
                alturas[i] = alt
        
        step += INC_STEP

def incr_a():
    tmpx = posx
    tmpy = posy
    if posr in [1, 4, 7, 10]:
        posx = (3 * tmpy + 4 * tmpx) // 5
        posy = (4 * tmpy - 3 * tmpx) // 5
    elif posr in [2, 5, 8, 11]:
        posx = (tmpy + 2 * tmpx) // 2
        posy = (2 * tmpy - tmpx) // 2
    else:
        posx = (4 * tmpx + 2 * tmpy) // 5
        posy = (4 * tmpy - 2 * tmpx) // 5  # Corrected typo from source
    posr = (posr + 1) % 12

def decr_a():
    tmpx = posx
    tmpy = posy
    if posr in [1, 4, 7, 10]:
        posx = (2 * tmpx - tmpy) // 2
        posy = (2 * tmpy + tmpx) // 2
    elif posr in [2, 5, 8, 11]:
        posx = (-3 * tmpy + 4 * tmpx) // 5
        posy = (4 * tmpy + 3 * tmpx) // 5
    else:
        posx = (-2 * tmpy + 4 * tmpx) // 5
        posy = (4 * tmpy + 2 * tmpx) // 5
    posr = (posr - 1) % 12
```

### Edge Cases and Pitfalls
- **Fixed-Point Division**: Integer division `x/2` truncates toward zero, which may cause asymmetry in coordinate mapping. This is mitigated by masking results with `(dim_mapa - 1)` to wrap coordinates within the 1024x1024 map bounds [S1, unofficial].
- **Height Buffer Reset**: The height buffer `alturas` must be reset to 0 every frame; otherwise, previous frame's occlusion data will corrupt the current render [S1, unofficial].
- **VRAM Buffer Swapping**: Relies on `REG_DISPCNT_L^=BACK_BUFFER_FLAG` and pointer toggling. Failing to wait for VBLANK (`while (!((REG_DISPSTAT)&1))`) causes tearing [S1, unofficial].
- **Source Disagreement**: The source code snippets show variations in rotation cases. Implementers should verify the exact mapping for their specific `posr` values against the source code's switch statements [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of depth slices and $M$ is the screen width. Each pixel is processed once per slice [S1, unofficial].
- **Memory Complexity**: $O(M + N \cdot M)$ for lookup tables and height buffer. The heightmap data is $O(W^2)$ where $W$ is the map dimension [S1, unofficial].
- **Measured Performance**: Optimized for the Game Boy Advance's ARM7TDMI CPU. Uses precomputed lookup tables to minimize runtime calculations [S1, unofficial].
- **Scaling**: Performance scales linearly with screen width and number of depth slices [S1, unofficial].
- **Strengths**: Fast on constrained hardware, simple occlusion handling, easy to implement [S1, unofficial].
- **Limitations**: Limited to heightmap-based terrain, fixed resolution, potential for visual artifacts due to fixed-point math [S1, unofficial].

## Background
The GBA Voxel Slice Heightmap Renderer was developed for the Game Boy Advance (GBA) to render 3D voxel terrain efficiently on its limited hardware [S1, unofficial]. The technique leverages the GBA's ARM7TDMI CPU capabilities, using fixed-point arithmetic and precomputed lookup tables to optimize the rendering loop [S1, unofficial]. It is part of a broader effort to demonstrate 3D graphics capabilities on the GBA [S1, unofficial].

## Sources
[S1] juanmv94/GameBoy-Advance-3D-voxel-space-demo, https://github.com/juanmv94/GameBoy-Advance-3D-voxel-space-demo, fetched 2026-10-27 (unofficial)
