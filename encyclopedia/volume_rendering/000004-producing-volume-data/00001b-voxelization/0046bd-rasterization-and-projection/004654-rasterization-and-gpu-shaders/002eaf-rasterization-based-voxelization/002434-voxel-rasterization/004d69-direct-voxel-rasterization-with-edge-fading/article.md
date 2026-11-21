# Direct Voxel Rasterization with Edge Fading

Also called: Max-blend voxel rasterization, Volume Sphere Rasterization, Voxel Sphere, Voxel-based sphere rendering

## Overview
Direct Voxel Rasterization with Edge Fading renders 3D spheres into a voxel grid by iterating over axis-aligned bounding boxes, computing Euclidean distance to the sphere center, applying a linear edge fade, and blending colors using maximum blending [S2, unofficial]. It differs from standard binary voxelization by introducing anti-aliased edges via intensity falloff, and from ray-marching volume rendering by using a direct, iterative rasterization approach that avoids complex traversal structures [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of spheres, each defined by a 3D position $(c_x, c_y, c_z)$, a radius $r$, and an RGB color $(C_r, C_g, C_b)$ [S2, unofficial].
*   **Output:** A 4D voxel grid `raster.data` of dimensions $(L, H, W)$ indexed as `[z, y, x, channel]` [S2, unofficial].

**Data Structures:**
*   `raster.data`: A 4D array with memory layout `[z, y, x, channel]` [S2, unofficial].
    *   Channel 0 stores Red, Channel 1 stores Green, Channel 2 stores Blue [S2, unofficial].
    *   Dimensions correspond to `(length, height, width)` [S2, unofficial].

**Parameters:**
*   `RENDER_FADE_MARGIN`: A float controlling the width of the edge fade region relative to the sphere radius. Typical value: $0.2$ [S2, unofficial].

**Algorithm Steps:**
1.  **Sphere Validation:** If a sphere's radius $r \le 0.1$, skip it to avoid degenerate bounding boxes and unnecessary computation [S2, unofficial].
2.  **Bounding Box Calculation:** For each sphere with center $(c_x, c_y, c_z)$ and radius $r$, compute integer bounds clamped to grid boundaries:
    $$
    \begin{aligned}
    x_{min} &= \max(0, \lfloor c_x - r \rfloor) \\
    x_{max} &= \min(W - 1, \lfloor c_x + r \rfloor) \\
    y_{min} &= \max(0, \lfloor c_y - r \rfloor) \\
    y_{max} &= \min(H - 1, \lfloor c_y + r \rfloor) \\
    z_{min} &= \max(0, \lfloor c_z - r \rfloor) \\
    z_{max} &= \min(L - 1, \lfloor c_z + r \rfloor)
    \end{aligned}
    $$
    Clamping prevents out-of-bounds array access [S2, unofficial].
3.  **Voxel Iteration:** Iterate $z$ from $z_{min}$ to $z_{max}$, $y$ from $y_{min}$ to $y_{max}$, and $x$ from $x_{min}$ to $x_{max}$ [S2, unofficial].
4.  **Distance Calculation:** For each voxel $(x, y, z)$, compute deltas $dx = x - c_x$, $dy = y - c_y$, $dz = z - c_z$, then compute Euclidean distance $d = \sqrt{dx^2 + dy^2 + dz^2}$ [S2, unofficial].
5.  **Intensity and Color Calculation:**
    *   Define fade start threshold: $d_{fade} = r \times (1.0 - \text{RENDER\_FADE\_MARGIN})$ [S2, unofficial].
    *   If $d \le r$, set intensity $I = 1.0$ [S2, unofficial].
    *   Else if $d > d_{fade}$, set intensity $I = 1.0 - \frac{d - d_{fade}}{r \times \text{RENDER\_FADE\_MARGIN}}$ [S2, unofficial].
    *   Else, set $I = 0.0$ and skip blending [S2, unofficial].
    *   Compute new color components: $C'_r = \lfloor C_r \times I \rfloor$, $C'_g = \lfloor C_g \times I \rfloor$, $C'_b = \lfloor C_b \times I \rfloor$ [S2, unofficial].
6.  **Maximum Blending:** Update the voxel at `[z, y, x]` using maximum blending for each channel:
    $$
    \begin{aligned}
    \text{raster.data}[z, y, x, 0] &= \max(\text{raster.data}[z, y, x, 0], C'_r) \\
    \text{raster.data}[z, y, x, 1] &= \max(\text{raster.data}[z, y, x, 1], C'_g) \\
    \text{raster.data}[z, y, x, 2] &= \max(\text{raster.data}[z, y, x, 2], C'_b)
    \end{aligned}
    $$
    [S2, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def rasterize_spheres(spheres, raster_width, raster_height, raster_length, fade_margin=0.2):
    """
    Renders spheres into a voxel grid using direct rasterization with edge fading.
    
    Args:
        spheres: List of dicts with keys 'x', 'y', 'z', 'radius', 'color' (dict with 'r', 'g', 'b').
        raster_width: Width of the voxel grid (x-dimension).
        raster_height: Height of the voxel grid (y-dimension).
        raster_length: Length of the voxel grid (z-dimension).
        fade_margin: Controls the width of the edge fade region (default 0.2).
        
    Returns:
        A 4D list of shape (length, height, width, 3) with integer values.
    """
    # Initialize raster data with zeros. Layout: [z, y, x, channel]
    # Channel 0: Red, 1: Green, 2: Blue
    raster = [[[[0, 0, 0] for _ in range(raster_width)] 
               for _ in range(raster_height)] 
              for _ in range(raster_length)]
    
    for sphere in spheres:
        cx = sphere['x']
        cy = sphere['y']
        cz = sphere['z']
        r = sphere['radius']
        cr = sphere['color']['r']
        cg = sphere['color']['g']
        cb = sphere['color']['b']
        
        # Pitfall: Skip degenerate spheres
        if r <= 0.1:
            continue
            
        # Calculate bounding box and clamp to grid boundaries
        x_min = max(0, int(cx - r))
        x_max = min(raster_width - 1, int(cx + r))
        y_min = max(0, int(cy - r))
        y_max = min(raster_height - 1, int(cy + r))
        z_min = max(0, int(cz - r))
        z_max = min(raster_length - 1, int(cz + r))
        
        # Precompute fade start distance and denominator
        fade_start = r * (1.0 - fade_margin)
        fade_denominator = r * fade_margin
        
        # Iterate over bounding box
        for z in range(z_min, z_max + 1):
            for y in range(y_min, y_max + 1):
                for x in range(x_min, x_max + 1):
                    # Calculate distance
                    dx = x - cx
                    dy = y - cy
                    dz = z - cz
                    dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                    
                    # Calculate intensity
                    if dist <= r:
                        intensity = 1.0
                    elif dist > fade_start:
                        intensity = 1.0 - (dist - fade_start) / fade_denominator
                    else:
                        continue # Outside fade region, no contribution
                    
                    # Calculate new color values
                    new_r = int(cr * intensity)
                    new_g = int(cg * intensity)
                    new_b = int(cb * intensity)
                    
                    # Maximum blending
                    if new_r > raster[z][y][x][0]:
                        raster[z][y][x][0] = new_r
                    if new_g > raster[z][y][x][1]:
                        raster[z][y][x][1] = new_g
                    if new_b > raster[z][y][x][2]:
                        raster[z][y][x][2] = new_b
                        
    return raster
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R^3)$, where $N$ is the number of spheres and $R$ is the average radius in voxels, because each sphere iterates over its axis-aligned bounding box volume [G].
*   **Memory Complexity:** $O(L \cdot H \cdot W)$ for the voxel grid storage [S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with the number of spheres and cubically with their radius [G].
*   **Strengths:** Simple logic, no complex data structures required, produces smooth edges via fading [S2, unofficial].
*   **Limitations:** Maximum blending can cause color bleeding if not handled carefully; performance degrades quickly with large radii or high voxel counts; no support for transparency or alpha blending in the basic form [S2, unofficial].

## Background
This method originates from the `volumetric-display` project by fughilli, which targets rendering volumetric data for LED cube displays [S2, unofficial]. It combines basic geometric rasterization with anti-aliasing techniques (edge fading) to improve visual quality on discrete voxel grids [S2, unofficial].

## Sources
[S1] Multivox: Volumetric Display, https://github.com/AncientJames/multivox, fetched 2026-10-26 (unofficial)
[S2] fughilli/volumetric-display, https://github.com/fughilli/volumetric-display, fetched 2026-10-26 (unofficial)
