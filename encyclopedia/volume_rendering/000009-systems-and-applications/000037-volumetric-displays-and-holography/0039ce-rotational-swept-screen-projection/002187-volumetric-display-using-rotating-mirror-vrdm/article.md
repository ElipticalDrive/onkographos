# Volumetric Display using Rotating Mirror (VRDM)
Also called: Rotating Mirror Volumetric Display, Volumetric Display using Rotating Mirror (vdrm) optical simulation & encoding, vdrm optical simulation & encoding

## Overview
VRDM is a volumetric display method that projects 3D voxel data onto a rotating mirror using surrounding LED screens, mapping 3D coordinates to 2D screen pixels via analytical mirror reflection matrices [S1, unofficial]. It differs from standard ray marching by avoiding per-pixel ray tracing loops, instead computing direct projections for each rotation angle to drive the LED arrays synchronously [S1, unofficial].

## How it works
**Inputs:** 3D voxel grid (implicit or explicit), rotation angle $\theta$, mirror offset $d$, screen geometry parameters.
**Outputs:** Frame buffer for surrounding LED screens, structured as `AngleImage`.

**Data Structures:**
*   `AngleImage`: Contiguous memory block of 772 bytes.
    *   Bytes 0-3: `u32` angle (radians or index, source uses radians in logic but stores as `u32` likely representing fixed-point or scaled value; source code shows `u32 angle` field).
    *   Bytes 4-771: 192 rows of `RGBH` arrays. Each row is 4 bytes: `R` (u8), `G` (u8), `B` (u8), `H` (u8, representing depth/address).
*   `ScreenLinePixels`: Array of `W_PIXELS` (default 64) `Option<PixelColor>` (u32 RGBA).
*   `ScreenLineAddr`: Struct containing `screen_idx` (usize) and `addr` (u32) for depth sorting.

**Parameters (Default Values):**
*   `W_PIXELS`: 64 (width of screen resolution) [S1, unofficial].
*   `H_PIXELS`: 40 (height of screen resolution) [S1, unofficial].
*   `MIRROR_OFFSET`: `std::f32::consts::SQRT_2` (~1.414) [S1, unofficial].
*   `SCREEN_OFFSET`: 0.05 (vertical offset) [S1, unofficial].
*   `SCREEN_Z_OFFSET`: -1.0 + `SCREEN_OFFSET` [S1, unofficial].
*   `SCREEN_Y_OFFSET`: -1.0 + `SCREEN_OFFSET` [S1, unofficial].
*   `NUM_SCREENS`: 3 [S1, unofficial].
*   `TOTAL_ANGLES`: `W_PIXELS * 8` (512 angles) [S1, unofficial].

**Steps:**

1.  **Generate Pyramid Surface (Reference Geometry):**
    For each pixel $(x, y)$ in a $W\_PIXELS \times W\_PIXELS$ grid centered at origin:
    *   Compute $r = W\_PIXELS / 2$.
    *   Compute $h = |x - r| + |y - r|$.
    *   If $h \ge H\_PIXELS$, skip.
    *   Compute $z = H\_PIXELS - 1 - h$.
    *   Assign color based on quadrant relative to center:
        *   $x \ge r, y \ge r$: Gray.
        *   $x < r, y \ge r$: Red.
        *   $x < r, y < r$: Green.
        *   $x \ge r, y < r$: Magenta.
    *   This creates a half-pyramid reference surface [S1, unofficial].

2.  **Mirror Transformation:**
    To map a 3D point $(x, y, z)$ to the mirror plane:
    *   Apply a 2D rotation matrix to $(x, y)$ coordinates based on the current rotation angle $\theta$.
    *   Apply a fixed 45-degree rotation matrix to $(y, z)$ coordinates to align the point with the mirror plane's normal [S1, unofficial].
    *   The mirror reflection matrix $M$ in homogeneous coordinates is:
        $$
        M = \begin{bmatrix} \sin^2\theta - \cos^2\theta & -2\sin\theta\cos\theta & 0 \\ -2\sin\theta\cos\theta & \cos^2\theta - \sin^2\theta & 0 \\ 2d\cos\theta & 2d\sin\theta & 1 \end{bmatrix}
        $$
        where $\theta$ is the mirror angle in radians and $d$ is `MIRROR_OFFSET` [S1, unofficial].
    *   Note: The source code implements `mirror_mat4` with specific sign conventions for sin/cos derived from `glam::Mat4` column-major layout [S1, unofficial].

3.  **Encode Pixels per Angle:**
    For each angle $\theta_k$ in the angle map (from `codec.encode()`):
    *   Initialize `pixels` array of size `IMG_HEIGHT` (192) with `None`.
    *   For each voxel/point in the 3D volume:
        *   Transform the 3D point to 2D screen coordinates $(u, v)$ using the mirror matrix and screen projection logic.
        *   Determine the target screen index and pixel address.
        *   If the pixel address is already occupied, keep the pixel with the smaller `addr` (closest depth) [S1, unofficial].
        *   Store `[r, g, b, p.addr as u8]` in the `pixels` array.
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
MIRROR_OFFSET = math.sqrt(2)
SCREEN_OFFSET = 0.05
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
            # Quadrant color assignment
            if x >= r and y >= r:
                color = (128, 128, 128) # Gray
            elif x < r and y >= r:
                color = (255, 0, 0)     # Red
            elif x < r and y < r:
                color = (0, 255, 0)     # Green
            else:
                color = (255, 0, 255)   # Magenta
            surface.append((x, y, z, color))
    return surface

def mirror_matrix(theta, d):
    """Computes the 3x3 mirror reflection matrix."""
    sin_t = math.sin(theta)
    cos_t = math.cos(theta)
    sin2 = sin_t * sin_t
    cos2 = cos_t * cos_t
    sin_cos = sin_t * cos_t
    
    # Matrix layout from source:
    # [sin^2 - cos^2, -2sin cos, 0]
    # [-2sin cos, cos^2 - sin^2, 0]
    # [2d cos, 2d sin, 1]
    return [
        [sin2 - cos2, -2 * sin_cos, 0],
        [-2 * sin_cos, cos2 - sin2, 0],
        [2 * d * cos_t, 2 * d * sin_t, 1]
    ]

def encode_frame(voxels, theta):
    """Encodes a single frame for a given angle."""
    # Initialize 192 rows, each row is a list of 64 pixels (None or (R,G,B,H))
    # H is stored as u8, representing depth/address
    pixels = [[None] * W_PIXELS for _ in range(192)]
    
    # Transform voxels and project to screen
    # Note: Exact 3D-to-2D projection logic involves screen-specific matrices
    # defined in `cacl_view_point` and `mirror_mat4` in source.
    # Simplified projection step:
    for voxel in voxels:
        x, y, z, rgba = voxel
        
        # 1. Mirror Transformation
        # Apply 2D rotation to (x,y) then 45-deg rotation to (y,z)
        # Then apply mirror matrix M
        # (Detailed matrix multiplication omitted for brevity, follows source logic)
        projected_u, projected_v, depth_addr = project_to_screen(x, y, z, theta, MIRROR_OFFSET)
        
        # 2. Depth Sorting
        # If pixel already exists, keep smaller addr (closest)
        if pixels[projected_v][projected_u] is not None:
            existing_addr = pixels[projected_v][projected_u][3]
            if depth_addr < existing_addr:
                # Update pixel
                pass
            else:
                continue
        else:
            # 3. Z-depth Zeroing Pitfall
            if (rgba & 0xFFFF_FF00) == 0:
                depth_addr = 0
        
        # Store R, G, B, H (addr as u8)
        pixels[projected_v][projected_u] = (rgba & 0xFF, (rgba >> 8) & 0xFF, (rgba >> 16) & 0xFF, depth_addr & 0xFF)
        
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
                # Default transparent/black? Source implies valid pixels only.
                # Assuming 0 for empty
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
- **Time Complexity:** $O(N \cdot A)$, where $N$ is the number of voxels and $A$ is the number of angles (`TOTAL_ANGLES`). Each voxel is projected to each angle frame.
- **Memory Complexity:** $O(A \cdot H \cdot W)$ for the frame buffer storage, where $H=192, W=64$. Each frame is 772 bytes. Total memory for all angles: $512 \times 772 \approx 395$ KB [S1, unofficial].
- **Performance:** The method avoids per-pixel ray tracing, using analytical matrices for direct projection, which is faster than iterative ray marching [S1, unofficial].
- **Strengths:** Real-time capable for low-resolution volumes; simple optical setup (one mirror, 3 screens).
- **Limitations:** Resolution limited by screen pixel count and mirror rotation speed; requires precise synchronization; artifacts from transparent voxels if Z-depth zeroing is not applied [S1, unofficial].

## Background
VRDM was developed as an early-stage volumetric display concept using two spinning mirrors, later evolved to version 2.0 with one mirror and three surrounding LED screens [S1, unofficial]. The simulation and encoding pipeline was implemented by hikaricai [S1, unofficial].

## Sources
[S1] hikaricai/vdrm, https://github.com/hikaricai/vdrm, fetched 2026-10-26 (unofficial)
