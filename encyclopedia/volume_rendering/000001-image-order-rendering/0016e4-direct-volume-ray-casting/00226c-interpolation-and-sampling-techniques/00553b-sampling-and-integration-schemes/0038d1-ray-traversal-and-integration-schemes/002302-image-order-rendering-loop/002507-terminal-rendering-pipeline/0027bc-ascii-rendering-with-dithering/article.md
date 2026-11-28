# ASCII Rendering with Dithering

Also called: ASCII Ray Tracing, ASCII Tracing with Ordered Dithering, Terminal Ray Tracing, Terminal Raymarching, VT100 Rendering

## Overview
ASCII Rendering with Dithering renders 3D scenes to a text-based terminal buffer by casting rays, computing lighting, and mapping the resulting continuous-tone color to a limited set of ANSI escape codes and ASCII characters [S1, unofficial]. It distinguishes itself from standard terminal rendering by using ordered Bayer matrix dithering to simulate grayscale intensity and multi-sampling anti-aliasing (MSAA) to reduce jagged edges, effectively bridging the gap between discrete character cells and continuous image perception [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scene defined by Signed Distance Functions (SDFs) for primitives (spheres, tori, planes), camera parameters (position, orientation), and light parameters (position, color).
*   **Output:** A 80x24 grid of ANSI escape sequences and ASCII characters printed to `stdout`.

**Data Structures:**
1.  **Bayer Matrix ($B$):** A $4 \times 4$ integer array storing values 1–16, used for dithering offsets. Stored in row-major order:
    $$
    B = \begin{bmatrix}
    1 & 9 & 3 & 11 \\
    13 & 5 & 15 & 7 \\
    4 & 12 & 2 & 10 \\
    16 & 8 & 14 & 6
    \end{bmatrix}
    $$
2.  **Color Lookup Table (LUT):** A precomputed table `color_LUT[16*16*16]` mapping quantized RGB components (0–15) to a combined foreground/background ANSI index. The index is calculated as `bg * 32 + fg`, where `fg` is the 5-bit foreground index and `bg` is the 3-bit background index [S1, unofficial].
3.  **Material Properties:**
    *   `mcol`: Array of `vec3` diffuse colors.
    *   `mshiny`: Array of `float` shininess exponents.
4.  **Camera Space Basis:**
    *   `camz`: Normalized vector from camera position to origin (or light).
    *   `camx`: Normalized cross product of `camz` and world up `(0,1,0)`.
    *   `camy`: Normalized cross product of `camx` and `camz`.

**Parameters:**
*   **Resolution:** 80 columns ($x \in [0, 79]$), 24 rows ($y \in [0, 23]$) [S1, unofficial].
*   **Samples per Pixel:** 8 (2 samples in X, 4 samples in Y) [S1, unofficial].
*   **Light Color:** $L_{col} = \{0.7, 0.5, 0.5\}$ [S1, unofficial].
*   **Shininess:** $\{10, 10, 10\}$ for materials 0, 1, 2 respectively [S1, unofficial].
*   **Material Colors:**
    *   Material 0: $\{0.0, 1.6, 0.0\}$ (Green) [S1, unofficial].
    *   Material 1: $\{0.0, 0.0, 0.0\}$ (Black) [S1, unofficial].
    *   Material 2: $\{1.0, 0.0, 0.0\}$ (Red) [S1, unofficial].
    *   *Note:* Other examples use Material 0 as floor $\{1.5, 1.5, 1.5\}$ and Material 2 as logo $\{0.0, 0.4, 1.0\}$ [S1, unofficial].
*   **Gamma Correction:** Exponent 2.2 is mentioned as a parameter, though the core loop implementation applies linear accumulation followed by quantization [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Call `render_init()` to populate the `color_LUT`. For each RGB component $c \in [0, 15]$, compute the nearest ANSI foreground (`fg`) and background (`bg`) indices using a brute-force search over 17 palette entries (0–16) minimizing squared Euclidean distance in RGB space [S1, unofficial].
2.  **Frame Loop:** For each frame:
    a.  Move cursor up 24 lines to overwrite previous output.
    b.  Iterate $y$ from 0 to 23.
    c.  Iterate $x$ from 0 to 79.
    d.  **Ray Construction:**
        i.  Initialize color accumulator $C = \{0, 0, 0\}$.
        ii. Perform MSAA by iterating offsets $xx \in \{-0.25, 0.25\}$ and $yy \in \{-0.75, -0.25, 0.25, 0.75\}$.
        iii. Construct ray direction in screen space:
        $$
        p_{screen} = (x - 40.0 + xx, \quad 25.0 - 2.0 \cdot y + yy, \quad 40.0)
        $$
        iv. Normalize $p_{screen}$ to get $dir_{screen}$.
        v. Transform to camera space:
        $$
        dir = camx \cdot dir_{screen}.x + camy \cdot dir_{screen}.y + camz \cdot dir_{screen}.z
        $$
    e.  **Sphere Tracing:**
        i.  Initialize $t = 0$, material $m = -1$.
        ii. Loop up to 64 iterations while $t < 10^6$:
        iii. Compute point $P = dir \cdot t + camPos$.
        iv. Query SDF: $d = dist(P, m)$.
        v. If $d < 0.001$:
            - Compute lighting $L = lighting(P, m, lightPos)$.
            - Add to accumulator: $C = C + L$.
            - Break loop.
        vi. Else: $t += d$.
    f.  **Accumulation:**
        i.  After 8 samples, scale color: $C = C \cdot 0.125$.
    g.  **Dithering and Color Mapping:**
        i.  Calculate Bayer offset:
        $$
        o = -48 + 4 \cdot B[(x \bmod 4) + (y \bmod 4) \cdot 4]
        $$
        *Note: The source code uses `bayer_matrix[(x&3) + (y&3)*4]*(1.0/8.0)` and subtracts this from the scaled component before clamping to 0-15. The formula in the notes $o = -48 + ...$ appears to be a derived constant for a different scaling range. The implementation logic is:*
        $$
        r = \text{clamp}(\lfloor C.x \cdot 15.0 - o \rfloor, 0, 15)
        $$
        $$
        g = \text{clamp}(\lfloor C.y \cdot 15.0 - o \rfloor, 0, 15)
        $$
        $$
        b = \text{clamp}(\lfloor C.z \cdot 15.0 - o \rfloor, 0, 15)
        $$
        ii. Lookup ANSI code: `code = color_LUT[r + g*16 + b*256]`.
        iii. Extract `fg = code & 31`, `bg = code >> 5`.
    h.  **Output:**
        i.  If `fg >= 0`: Print ANSI sequence for foreground color `30 + (fg & 7)` and bold flag `!!(fg & 8)`, followed by character `'#'`.
        ii. Else: Print space `' '`.
        iii. Handle background color changes efficiently by only printing `\x1b[40+bg` if `bg` changes from previous pixel.
    i.  After row $y=23$, print newline and reset attributes.
3.  **Refresh:** Flush stdout, sleep 20ms, increment frame counter.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import time
import sys

# Constants
WIDTH = 80
HEIGHT = 24
BAYER_MATRIX = [
    [1, 9, 3, 11],
    [13, 5, 15, 7],
    [4, 12, 2, 10],
    [16, 8, 14, 6]
]

# Material definitions (example from notes)
MATERIAL_COLORS = [
    [0.0, 1.6, 0.0],  # Green
    [0.0, 0.0, 0.0],  # Black
    [1.0, 0.0, 0.0]   # Red
]
SHININESS = [10, 10, 10]
LIGHT_COLOR = [0.7, 0.5, 0.5]

def nearest_color(rgb):
    """
    Finds the best ANSI foreground (fg) and background (bg) indices.
    Simplified: returns a combined index for LUT lookup.
    In practice, this requires the full palette search described in [S1, unofficial].
    Here we assume a helper exists or the LUT is precomputed.
    """
    # Placeholder for the brute-force search logic from [S1, unofficial]
    # Returns (fg, bg)
    return 7, 0 

def render_init():
    """Precompute color LUT."""
    # In the actual C implementation, this iterates r,g,b 0..15
    # and calls nearest_color. We assume the LUT is available.
    pass

def dist(p, material_out):
    """
    Signed Distance Function for the scene.
    Returns distance and sets material_out.
    """
    # Example: Plane and Torus from [S1, unofficial]
    # Plane at y = -50
    d_plane = p[1] + 50.0
    m_plane = 0 # Material 0
    
    # Torus
    # Rotate point for animation
    frame = 0.0 # Simplified
    ca = math.cos(frame * 0.025)
    sa = math.sin(frame * 0.025)
    cx = p[0] * ca - p[2] * sa
    cz = p[0] * sa + p[2] * ca
    
    # Rotate Y
    cb = math.cos(frame * 0.07)
    sb = math.sin(frame * 0.07)
    qx = cx * cb + p[1] * sb
    qy = -cx * sb + p[1] * cb
    qz = cz
    
    # Torus SDF: length(length(q.xy) - 50) - 20
    len_xy = math.sqrt(qx**2 + qy**2)
    d_torus = math.sqrt((len_xy - 50)**2 + qz**2) - 20
    m_torus = 2 # Material 2
    
    if d_torus < d_plane:
        material_out[0] = m_torus
        return d_torus
    else:
        material_out[0] = m_plane
        return d_plane

def lighting(p, m, light_pos):
    """Compute diffuse and specular lighting."""
    light_dir = [light_pos[i] - p[i] for i in range(3)]
    l_len = math.sqrt(sum(c**2 for c in light_dir))
    light_dir = [c/l_len for c in light_dir]
    
    # Approximate normal using SDF gradient
    eps = 0.1
    mat_tmp = [-1]
    d_near = dist([p[i] + light_dir[i]*eps for i in range(3)], mat_tmp)
    normal = [dist([p[i] - light_dir[i]*eps for i in range(3)], mat_tmp) - d_near for i in range(3)]
    # Normalize normal
    n_len = math.sqrt(sum(c**2 for c in normal))
    if n_len > 0:
        normal = [c/n_len for c in normal]
        
    # Diffuse
    diff = max(0.0, sum(normal[i]*light_dir[i] for i in range(3)))
    
    # Shadow (simplified)
    shadow = 1.0
    # ... shadow casting logic omitted for brevity, assumed 1.0 for basic demo
    
    l = max(0.1, diff * shadow)
    
    # Material color
    mat_col = MATERIAL_COLORS[m]
    
    # Specular
    # View vector is -ray_dir, but in this simple demo, view is often camera pos
    # Using a simplified specular term as per notes: lcol * pow(l, shiny)
    spec = LIGHT_COLOR[i] * (l ** SHININESS[m]) # Note: This formula in notes is simplified
    
    color = [mat_col[i]*l + LIGHT_COLOR[i]*(l**SHININESS[m]) for i in range(3)]
    return color

def render_frame(frame_count):
    render_init()
    
    # Camera setup
    cam_pos = [120*math.sin(frame_count*0.01), 40 + 30*math.sin(frame_count*0.03), -120*math.cos(frame_count*0.01)]
    cam_z = [-c/math.sqrt(sum(c**2 for c in cam_pos)) for c in cam_pos] # Normalize
    cam_x = [0,1,0] # World up
    # Cross product cam_z x cam_x
    cross_x = [cam_z[1]*cam_x[2]-cam_z[2]*cam_x[1], 
               cam_z[2]*cam_x[0]-cam_z[0]*cam_x[2], 
               cam_z[0]*cam_x[1]-cam_z[1]*cam_x[0]]
    cam_x = [c/math.sqrt(sum(c**2 for c in cross_x)) for c in cross_x]
    # Cross product cam_x x cam_z
    cross_y = [cam_x[1]*cam_z[2]-cam_x[2]*cam_z[1],
               cam_x[2]*cam_z[0]-cam_x[0]*cam_z[2],
               cam_x[0]*cam_z[1]-cam_x[1]*cam_z[0]]
    cam_y = [c/math.sqrt(sum(c**2 for c in cross_y)) for c in cross_y]
    
    light_pos = [0, 40, 0] # Example light pos
    
    # ANSI escape codes
    sys.stdout.write("\x1b[24A") # Move cursor up 24 lines
    
    for y in range(HEIGHT):
        for x in range(WIDTH):
            color_acc = [0.0, 0.0, 0.0]
            
            # MSAA
            for xx in [-0.25, 0.25]:
                for yy in [-0.75, -0.25, 0.25, 0.75]:
                    # Ray direction
                    rx = x - 40.0 + xx
                    ry = 25.0 - 2.0 * y + yy
                    rz = 40.0
                    r_len = math.sqrt(rx**2 + ry**2 + rz**2)
                    dir_screen = [rx/r_len, ry/r_len, rz/r_len]
                    
                    # Transform to camera space
                    dir_cam = [
                        dir_screen[0]*cam_x[0] + dir_screen[1]*cam_y[0] + dir_screen[2]*cam_z[0],
                        dir_screen[0]*cam_x[1] + dir_screen[1]*cam_y[1] + dir_screen[2]*cam_z[1],
                        dir_screen[0]*cam_x[2] + dir_screen[1]*cam_y[2] + dir_screen[2]*cam_z[2]
                    ]
                    
                    # Sphere Tracing
                    t = 0.0
                    m = -1
                    hit = False
                    for _ in range(64):
                        if t > 1e6:
                            break
                        p = [cam_pos[i] + dir_cam[i]*t for i in range(3)]
                        mat_out = [-1]
                        d = dist(p, mat_out)
                        m = mat_out[0]
                        
                        if d < 0.001:
                            l = lighting(p, m, light_pos)
                            color_acc = [color_acc[i] + l[i] for i in range(3)]
                            hit = True
                            break
                        t += d
                    
                    if not hit:
                        # Background color (black)
                        pass
            
            # Scale for AA
            color_acc = [c * 0.125 for c in color_acc]
            
            # Dithering
            bayer_val = BAYER_MATRIX[(x & 3) + ((y & 3) * 4)]
            # Offset calculation from code: bayer_matrix[...] * (1.0/8.0)
            # But notes say: o = -48 + 4 * B[...]
            # The code does: (int)(c.x * 15.0 - o) where o is bayer/8.
            # Let's stick to the code logic:
            o = bayer_val / 8.0
            
            r = max(0, min(15, int(color_acc[0] * 15.0 - o)))
            g = max(0, min(15, int(color_acc[1] * 15.0 - o)))
            b = max(0, min(15, int(color_acc[2] * 15.0 - o)))
            
            # Lookup ANSI code (Simplified: assume nearest color returns 7,0 for black)
            fg, bg = nearest_color([r/15.0, g/15.0, b/15.0])
            
            # Print
            if fg >= 0:
                # Foreground color 30+(fg&7), bold !!(fg&8)
                # Background 40+bg
                # This part is complex in ANSI, simplified here to just '#'
                sys.stdout.write('#')
            else:
                sys.stdout.write(' ')
        sys.stdout.write('\n')
        
    sys.stdout.flush()
    time.sleep(0.02)

if __name__ == "__main__":
    frame = 0
    try:
        while True:
            render_frame(frame)
            frame += 1
    except KeyboardInterrupt:
        pass
```

## Subtopics

### ASCII Plot Visualization
A utility function that renders 2D image-like objects (e.g., projection slices) to standard output as ASCII art using a character ramp [S2, unofficial]. It calculates min/max intensity values, maps pixel intensities to a character string index using a square-root scaling function, and outputs the characters [S2, unofficial]. The character ramp is ordered from low to high intensity, typically starting with space and ending with dense characters like `@` or `#` [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(W \cdot H \cdot S \cdot I)$, where $W$ is width, $H$ is height, $S$ is samples per pixel, and $I$ is max sphere tracing iterations. For 80x24x8x64, this is roughly 98,304 SDF evaluations per frame.
*   **Memory Complexity:** $O(1)$ for the scene (if SDFs are analytic) plus $O(1)$ for the LUT (16KB).
*   **Performance:** Extremely slow on CPU. The example code is described as "pretty slow and unoptimized" [S1, unofficial]. It is suitable for educational purposes or obfuscated code contests, not real-time interactive rendering.
*   **Strengths:** No graphics hardware required; works in any terminal emulator; demonstrates ray marching concepts in a minimal codebase.
*   **Limitations:** Low resolution (80x24); limited color palette (16 colors + bold); no transparency or complex reflections; slow frame rates.

## Background
ASCII Rendering with Dithering originated as a technique to render 3D scenes in text-based terminals using VT100/ANSI escape codes [S1, unofficial]. It is often compared to Shadertoy demos but adapted for text output [S1, unofficial]. The technique was popularized by demos such as those by `a1k0n` (a1k0n/asciitracing), which rendered scenes like the Square logo using constructive solid geometry and sphere tracing [S1, unofficial]. The use of ordered dithering (Bayer matrix) allows for the simulation of grayscale intensity within the limited ANSI color palette [S1, unofficial].

## Sources
[S1] a1k0n/asciitracing, https://github.com/a1k0n/asciitracing, fetched 2026-10-26 (unofficial)
[S2] jwbuurlage/TPT, https://github.com/jwbuurlage/TPT, fetched 2026-10-26 (unofficial)
