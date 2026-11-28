# Procedural Island Generation

Also called: Island Heightmap Generation, Flare-Based Island Generation

## Overview
An algorithm to generate island heightmaps using flares, noise, erosion simulation, and slope-based coloring [S1, unofficial]. It ensures the generated terrain is surrounded by water by applying a radial distance-based multiplier, then enhances natural features through noise multiplication and power-law exaggeration. It differs from general terrain generation by enforcing a closed-water boundary condition and using slope-aware gradient sampling for coloring [S1, unofficial].

## How it works
**Inputs:**
- Grid resolution $N \times N$ (square grid).
- Island center $(c_x, c_y)$, typically $(0.5, 0.5)$ in normalized coordinates $[0, 1]$.
- Island radius $R$ (normalized, typically $0.5$).
- Flare power $f > 0$.
- Noise function $Noise(x, y)$ (e.g., cubic, Perlin, or simplex noise with octaves).
- Exaggeration multiplier $m > 1$.
- Exaggeration power $p > 1$.
- Slope influence power $x \in [0, 1]$.
- Color gradient function $Gradient(h)$ mapping height $h \in [0, 1]$ to color.

**Outputs:**
- Heightmap $H$ of size $N \times N$ with values in $[0, 1]$ (or slightly above, capped later).
- Colored island image.

**Data Structures:**
- 2D array `heightmap[N][N]` of floats.
- 2D array `normal[N][N]` of 3D vectors (for slope calculation).

**Steps:**

1. **Initialize Heightmap:** Create a $N \times N$ grid initialized to zeros.

2. **Apply Flare:** For each pixel $(i, j)$ at normalized coordinates $(u, v) \in [0, 1]^2$:
   - Calculate distance to center: $d = \sqrt{(u - c_x)^2 + (v - c_y)^2}$.
   - Normalize distance: $d_{norm} = \min(1.0, d / R)$.
   - Calculate flare value:
     $$ F = \left(\frac{\cos(\pi \cdot d_{norm}) + 1}{2}\right)^f $$
   - Set `heightmap[i][j] = F`.
   - *Note:* This ensures height is 1 at the center and 0 at the boundary $R$, creating a water surround [S1, unofficial].

3. **Add Noise:** For each pixel $(i, j)$:
   - Sample noise: $n = Noise(u, v)$.
   - Multiply: `heightmap[i][j] = heightmap[i][j] * n`.
   - *Note:* Coherent noise adds natural variation. Values may become negative; clamp to 0 if necessary, though the source implies multiplication preserves the shape [S1, unofficial].

4. **Exaggerate Features:** For each pixel $(i, j)$:
   - Let $h = \text{heightmap}[i][j]$.
   - Apply power and multiplier: $h' = m \cdot h^p$.
   - Set `heightmap[i][j] = h'`.
   - *Note:* Since $h \in [0, 1]$ and $p > 1$, valleys shrink more than peaks, exaggerating mountains. The multiplier $m > 1$ restores overall height scale [S1, unofficial].

5. **Cap Heights:** For each pixel $(i, j)$:
   - If `heightmap[i][j] > 1.0`, set `heightmap[i][j] = 1.0`.
   - *Note:* This prevents values exceeding the gradient range, though the source mentions lowering plateaus by subtracting $h-1$; capping is the standard interpretation for heightmap normalization [S1, unofficial].

6. **Compute Normals:** For each pixel $(i, j)$:
   - Calculate surface normal $\vec{n}$ using finite differences from the heightmap.
   - Let $\vec{u} = (0, 1, 0)$ (up vector).
   - Compute dot product: $dot = \vec{n} \cdot \vec{u}$.
   - *Note:* $\vec{n} \cdot \vec{u}$ ranges from 0 (vertical wall) to 1 (flat horizontal). Steep slopes have low dot products [S1, unofficial].

7. **Slope-Based Coloring:** For each pixel $(i, j)$:
   - Let $h = \text{heightmap}[i][j]$.
   - Calculate modified height for gradient sampling:
     $$ h_{mod} = \left(h^{(\vec{n} \cdot \vec{u})}\right)^x $$
   - Sample color: $C = Gradient(h_{mod})$.
   - *Note:* If the slope is steep ($\vec{n} \cdot \vec{u}$ is small), $h_{mod}$ approaches 1, sampling higher up the gradient (e.g., rock color) even at lower altitudes. Lowering $x$ reduces this effect [S1, unofficial].

8. **Optional Plateau Reduction:** If flat tops are undesirable:
   - For each pixel where $h > 1.0$ (before capping in step 5, or if capping was skipped):
     - $h = h - (h - 1) = 1.0$.
   - *Note:* The source describes subtracting $h-1$ to lower plateaus, which effectively caps at 1.0. If a volcano gradient is used, this step might be skipped or modified [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def procedural_island_generation(N, f, m, p, x, noise_func, gradient_func):
    """
    Generate an island heightmap and color it.
    
    Args:
        N: Grid resolution (N x N).
        f: Flare power.
        m: Exaggeration multiplier (> 1).
        p: Exaggeration power (> 1).
        x: Slope influence power ([0, 1]).
        noise_func: Function noise_func(u, v) returning coherent noise.
        gradient_func: Function gradient_func(h) returning color for height h.
    
    Returns:
        colored_image: N x N x 3 array of colors.
    """
    heightmap = np.zeros((N, N))
    
    # Step 2: Apply Flare
    for i in range(N):
        for j in range(N):
            u = i / (N - 1)
            v = j / (N - 1)
            # Distance to center (0.5, 0.5)
            d = math.sqrt((u - 0.5)**2 + (v - 0.5)**2)
            d_norm = min(1.0, d / 0.5) # R=0.5
            
            # Flare formula
            flare = ((math.cos(math.pi * d_norm) + 1) / 2) ** f
            heightmap[i, j] = flare
            
    # Step 3: Add Noise
    for i in range(N):
        for j in range(N):
            u = i / (N - 1)
            v = j / (N - 1)
            noise_val = noise_func(u, v)
            heightmap[i, j] *= noise_val
            
    # Step 4: Exaggerate Features
    for i in range(N):
        for j in range(N):
            h = heightmap[i, j]
            # Clamp h to [0, 1] before power to avoid complex results if negative
            h = max(0.0, min(1.0, h))
            heightmap[i, j] = m * (h ** p)
            
    # Step 5: Cap Heights
    heightmap = np.clip(heightmap, 0.0, 1.0)
    
    # Step 6: Compute Normals
    # Simple finite difference for normals
    normals = np.zeros((N, N, 3))
    for i in range(1, N-1):
        for j in range(1, N-1):
            h_left = heightmap[i, j-1]
            h_right = heightmap[i, j+1]
            h_down = heightmap[i-1, j]
            h_up = heightmap[i+1, j]
            
            # Gradient in x and y
            dx = h_right - h_left
            dy = h_up - h_down
            
            # Normal is (-dx, -dy, 1) normalized
            n = np.array([-dx, -dy, 1.0])
            n = n / np.linalg.norm(n)
            normals[i, j] = n
            
    # Step 7: Slope-Based Coloring
    colored_image = np.zeros((N, N, 3))
    u_vec = np.array([0.0, 1.0, 0.0]) # Up vector
    
    for i in range(N):
        for j in range(N):
            h = heightmap[i, j]
            
            # Get normal, default to up if at boundary
            n = normals[i, j]
            if i == 0 or i == N-1 or j == 0 or j == N-1:
                n = u_vec
                
            dot = np.dot(n, u_vec)
            
            # Slope-based gradient sampling point
            # h_mod = (h ^ (n . u)) ^ x
            # Avoid 0^0 issues
            if h == 0:
                h_mod = 0.0
            else:
                h_mod = (h ** dot) ** x
                
            colored_image[i, j] = gradient_func(h_mod)
            
    return colored_image
```

**Parameters:**
- $f$: Controls island edge sharpness. Increasing $f$ makes the island more pointy; decreasing it increases surface area [S1, unofficial].
- $m$: Multiplier $> 1$ to restore height after power law reduction [S1, unofficial].
- $p$: Power $> 1$ to exaggerate mountains [S1, unofficial].
- $x$: Slope influence power in $[0, 1]$. Lowering $x$ reduces the influence of terrain slope on gradient sampling [S1, unofficial].

**Edge Cases:**
- **Negative Noise:** If noise is negative, the flare multiplication results in negative heights. The source does not explicitly state clamping, but heightmaps are typically non-negative. Clamping to 0 is recommended [G].
- **Boundary Normals:** Normals at grid boundaries are undefined. Defaulting to the up vector or mirroring is necessary [G].
- **Zero Height:** $0^{\text{dot}}$ is 0, which is safe. However, if $h=0$ and dot=0, $0^0$ is undefined. Handle explicitly [G].

## Operating characteristics
- **Time Complexity:** $O(N^2)$ for grid traversal. Noise evaluation and normal computation are constant time per pixel.
- **Memory Complexity:** $O(N^2)$ for heightmap and normal storage.
- **Strengths:** Simple to implement, produces visually pleasing island shapes with water boundaries, computationally inexpensive [S1, unofficial].
- **Limitations:** Lacks geological realism (erosion is simulated via power law, not physical simulation), plateau artifacts may require additional processing [S1, unofficial].
- **When to Use:** Quick preview generation, low-tech platforms, overview rendering of larger areas [S1, unofficial].
- **When Not to Use:** When realistic erosion patterns or complex geological features are required [S1, unofficial].

## Background
Originated in the context of layered voxel rendering for quick preview purposes on low-tech platforms [S1, unofficial]. The technique was described by Job Talle in "Layered voxel rendering" [S1, unofficial]. It is used for generating island heightmaps that are then decorated with trees and huts [S1, unofficial].

## Sources
[S1] Job Talle | Layered voxel rendering, https://jobtalle.com/layered_voxel_rendering.html, fetched 2026-10-28 (unofficial)
