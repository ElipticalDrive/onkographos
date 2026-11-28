# Volumetric Display using Rotating Mirror (VRDM)
Also called: Rotating Mirror Volumetric Display, Volumetric Display using Rotating Mirror (vdrm) optical simulation & encoding, vdrm optical simulation & encoding

## Overview
VRDM is a volumetric display method that projects 3D voxel data onto a rotating mirror using surrounding LED screens, mapping 3D coordinates to 2D screen pixels via analytical mirror reflection matrices [S1, unofficial]. It differs from standard ray marching by avoiding per-pixel ray tracing loops, instead computing direct projections for each rotation angle to drive the LED arrays synchronously [S1, unofficial].

## How it works
**Inputs:** 3D voxel grid (implicit or explicit), rotation angle $\theta$, mirror offset $d$, screen geometry parameters [S1, unofficial].
**Outputs:** Frame buffer for surrounding LED screens, structured as `AngleImage` [S1, unofficial].

**Data Structures:**
*   `AngleImage`: Contiguous memory block of 772 bytes [S1, unofficial].
    *   Bytes 0-3: `u32` angle value [S1, unofficial].
    *   Bytes 4-771: 192 rows of `RGBH` arrays [S1, unofficial]. Each row is 4 bytes: `R` (u8), `G` (u8), `B` (u8), `H` (u8 representing depth/address) [S1, unofficial].
*   `ScreenLinePixels`: Array of `W_PIXELS` `Option<PixelColor>` (u32 RGBA) [S1, unofficial].
*   `ScreenLineAddr`: Struct containing `screen_idx` (usize) and `addr` (u32) for depth sorting [S1, unofficial].

**Parameters (Default Values):**
*   `W_PIXELS`: 64 (screen width resolution) [S1, unofficial].
*   `H_PIXELS`: 40 (screen height resolution) [S1, unofficial].
*   `MIRROR_OFFSET`: 1.0 per notes, but source code uses $\sqrt{2} \approx 1.414$ [S1, unofficial].
*   `SCREEN_OFFSET`: 0.25 per notes, but source code uses 0.05 [S1, unofficial].
*   `SCREEN_X_OFFSET`: -0.75 [S1, unofficial].
*   `SCREEN_Z_OFFSET`: -1.0 + `SCREEN_OFFSET` [S1, unofficial].
*   `SCREEN_Y_OFFSET`: -1.0 + `SCREEN_OFFSET` [S1, unofficial].
*   `NUM_SCREENS`: 3 [S1, unofficial].
*   `TOTAL_ANGLES`: `W_PIXELS * 8` = 512 angles [S1, unofficial].

**Steps:**

1.  **Generate Pyramid Surface (Reference Geometry):**
    For each pixel $(x, y)$ in a $W\_PIXELS \times W\_PIXELS$ grid:
    *   Compute $r = W\_PIXELS / 2$.
    *   Compute $h = |x - r| + |y - r|$.
    *   If $h \ge H\_PIXELS$, skip.
    *   Compute $z = H\_PIXELS - 1 - h$.
    *   Assign color based on quadrant relative to center $(r, r)$: $x \ge r, y \ge r \to$ gray; $x < r, y \ge r \to$ red; $x < r, y < r \to$ green; $x \ge r, y < r \to$ magenta [S1, unofficial].
    *   This creates a half-pyramid reference surface [S1, unofficial].

2.  **Mirror Transformation:**
    To map a 3D point $(x, y, z)$ to the mirror plane:
    *   Apply a 2D rotation matrix to $(x, y)$ coordinates based on the current rotation angle $\theta$ [S1, unofficial].
    *   Apply a fixed 45-degree rotation matrix to $(y, z)$ coordinates to align the point with the mirror plane's normal [S1, unofficial].
    *   Apply the mirror reflection matrix $M$ in homogeneous coordinates:
        $$
        M = \begin{bmatrix} \sin^2\theta - \cos^2\theta & -2\sin\theta\cos\theta & 0 \\ -2\sin\theta\cos\theta & \cos^2\theta - \sin^2\theta & 0 \\ 2d\cos\theta & 2d\sin\theta & 1 \end{bmatrix}
        $$
        where $\theta$ is the mirror angle in radians and $d$ is `MIRROR_OFFSET` [S1, unofficial].
    *   Note: The source code implements `mirror_mat4` with specific sign conventions for sin/cos derived from column-major layout [S1, unofficial].

3.  **Encode Pixels per Angle:**
    For each angle $\theta_k$ in the angle map (from `codec.encode()`):
    *   Initialize `pixels` array of size `IMG_HEIGHT` (192) with `None` [S1, unofficial].
    *   For each voxel/point in the 3D volume:
        *   Transform the 3D point to 2D screen coordinates $(u, v)$ using the mirror matrix and screen projection logic [S1, unofficial].
        *   Determine the target screen index and pixel address [S1, unofficial].
        *   If the pixel address is already occupied, keep the pixel with the smaller `addr` (closest depth) [S1, unofficial].
        *   Store `[r, g, b, p.addr as u8]` in the `pixels` array [S1, unofficial].
    *   **Z-depth Zeroing Pitfall:** If the alpha channel of a pixel is zero (`rgba & 0xFFFF_FF00 == 0`), force the computed z-depth (or `H` value) to 0 to prevent rendering artifacts from transparent voxels [S1, unofficial].

4.  **Output Assembly:**
    Pack the `pixels` array into the `AngleImage` structure: `[angle(4), R(1), G(1), B(1), H(1), ...]` repeated 192 times [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants from source
W_PIXELS = 64
H_PIXELS = 40
IMG_HEIGHT = 192
MIRROR_OFFSET = math.sqrt(2)  # Source code uses sqrt(2); notes state 1.0
SCREEN_OFFSET = 0.05          # Source code uses 0.05; notes state 0.25
SCREEN_Z_OFFSET = -1.0 + SCREEN_OFFSET
SCREEN_Y_OFFSET = -1.0 + SCREEN_OFFSET
NUM_SCREENS = 3
TOTAL_ANGLES = W_PIXELS * 8

def generate_pyramid_surface():
    """Generates reference pyramid surface for encoding."""
    surface = []
    r = W_PIXELS / 2
    for x in range(W_PIXELS):
        for y in range(W_PIXELS):
            h = abs(x - r) + abs(y - r)
            if h >= H_PIXELS:
                continue
            z = H_PIXELS - 1 - h
            # Quadrant color assignment relative to center (r, r)
            if x >= r and y >= r:
                color = (128, 128, 128)  # Gray
            elif x < r and y >= r:
                color = (255, 0, 0)      # Red
            elif x < r and y < r:
                color = (0, 255, 0)      # Green
            else:
                color = (255, 0, 255)    # Magenta
            surface.append((x, y, z, color))
    return surface

def rotate_2d(x, y, theta):
    """Applies 2D rotation matrix to (x, y)."""
    cos_t = math.cos(theta)
    sin_t = math.sin(theta)
    return x * cos_t - y * sin_t, x * sin_t + y * cos_t

def rotate_yz_45(y, z):
    """Applies fixed 45-degree rotation matrix to (y, z)."""
    inv_sqrt2 = 1.0 / math.sqrt(2)
    return y * inv_sqrt2 - z * inv_sqrt2, y * inv_sqrt2 + z * inv_sqrt2

def apply_mirror_matrix(x, y, z, theta, d):
    """Applies the 3x3 mirror reflection matrix in homogeneous coordinates."""
    sin_t = math.sin(theta)
    cos_t = math.cos(theta)
    sin2 = sin_t * sin_t
    cos2 = cos_t * cos_t
    sin_cos = sin_t * cos_t
    # M * [x, y, z]^T
    x_new = (sin2 - cos2) * x - 2 * sin_cos * y
    y_new = -2 * sin_cos * x + (cos2 - sin2) * y
    z_new = 2 * d * cos_t * x + 2 * d * sin_t * y + z
    return x_new, y_new, z_new

def map_to_screen_coords(x, y, z):
    """Maps transformed coordinates to screen pixel indices (u, v).
    Exact scaling/offsets use the source's cacl_view_point logic [G]."""
    # Linear scaling placeholder for demonstration; replace with exact screen geometry math
    scale_x = W_PIXELS / 2.0
    scale_y = IMG_HEIGHT / 2.0
    u = int((x + 1.0) * scale_x)
    v = int((y + 1.0) * scale_y)
    return u, v

def encode_frame(voxels, theta):
    """Encodes a single frame for a given angle."""
    # Initialize 192 rows, each row is a list of 64 pixels (None or (R,G,B,H))
    pixels = [[None] * W_PIXELS for _ in range(IMG_HEIGHT)]
    
    for voxel in voxels:
        x, y, z, rgba = voxel
        
        # 1. Mirror Transformation
        x_rot, y_rot = rotate_2d(x, y, theta)
        y_rot, z_rot = rotate_yz_45(y_rot, z)
        x_proj, y_proj, z_proj = apply_mirror_matrix(x_rot, y_rot, z_rot, theta, MIRROR_OFFSET)
        
        # 2. Project to screen grid
        u, v = map_to_screen_coords(x_proj, y_proj, z_proj)
        
        # Bounds check
        if not (0 <= u < W_PIXELS and 0 <= v < IMG_HEIGHT):
            continue
            
        # Extract RGBA components
        r = rgba & 0xFF
        g = (rgba >> 8) & 0xFF
        b = (rgba >> 16) & 0xFF
        a = (rgba >> 24) & 0xFF
        
        # 3. Z-depth Zeroing Pitfall
        if (rgba & 0xFFFF_FF00) == 0:
            depth_addr = 0
        else:
            depth_addr = int(abs(z_proj) * 100) & 0xFF  # Scaled depth address as u8
            
        # 4. Depth Sorting
        if pixels[v][u] is not None:
            existing_addr = pixels[v][u][3]
            if depth_addr < existing_addr:
                pixels[v][u] = (r, g, b, depth_addr)
        else:
            pixels[v][u] = (r, g, b, depth_addr)
            
    return pixels

def assemble_angle_image(angle_val, pixels):
    """Assembles the 772-byte AngleImage structure."""
    buffer = bytearray(772)
    # Pack angle (4 bytes)
    buffer[0:4] = angle_val.to_bytes(4, byteorder='little')
    
    # Pack 192 rows of RGBH
    offset = 4
    for row in pixels:
        for pixel in row:
            if pixel is None:
                buffer[offset:offset+4] = bytes([0, 0, 0, 0])
            else:
                r, g, b, h = pixel
                buffer[offset:offset+4] = bytes([r, g, b, h])
            offset += 4
    return buffer
```

## Subtopics

### [Helical Display Surface Geometry & Balancing](8512-helical-display-surface-geometry-balancing.md)
Defines mechanical geometry and dynamic balancing for a rotating double-helix reflector. Distinguished by its double-helix topology minimizing drag/vibration, enabling "walk-around" viewing.

## Operating characteristics
- **Time Complexity:** $O(N \cdot A)$, where $N$ is the number of voxels and $A$ is the number of angles (`TOTAL_ANGLES`) [S1, unofficial]. Each voxel is projected to each angle frame [S1, unofficial].
- **Memory Complexity:** $O(A \cdot H \cdot W)$ for the frame buffer storage, where $H=192, W=64$ [S1, unofficial]. Each frame is 772 bytes [S1, unofficial].
- **Performance:** The method avoids per-pixel ray tracing, using analytical matrices for direct projection, which is faster than iterative ray marching [S1, unofficial].
- **Strengths:** Real-time capable for low-resolution volumes; simple optical setup (one mirror, 3 screens) [S1, unofficial].
- **Limitations:** Resolution limited by screen pixel count and mirror rotation speed; requires precise synchronization; artifacts from transparent voxels if Z-depth zeroing is not applied [S1, unofficial].

## Background
VRDM was developed as an early-stage volumetric display concept using two spinning mirrors, later evolved to version 2.0 with one mirror and three surrounding LED screens [S1, unofficial]. The simulation and encoding pipeline was implemented by hikaricai [S1, unofficial].

## Sources
[S1] hikaricai/vdrm, https://github.com/hikaricai/vdrm, fetched 2026-10-26 (unofficial)
