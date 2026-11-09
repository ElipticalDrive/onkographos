# Procedural Sphere Placement

Also called: Bubble Generation, Voxel Sphere Carving

## Overview
Procedural Sphere Placement populates a 3D voxel grid with spherical voids (density 0) inside a background field (density 255), using pre-sampled 2D coordinates and random radii [S1, unofficial]. It differs from its parent [Sphere Carving](2377-sphere-carving.md) by automating the generation of multiple, randomly sized and positioned spheres rather than carving a single defined sphere, and from [Voxel Deformation with Spherical Falloff](4413-voxel-deformation-with-spherical-falloff.md) by producing hard-edged binary voids rather than smooth density gradients [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Grid dimensions $(maxX, maxY, maxZ)$, sampling stride $sep$, and radius distribution parameters.
- Output: A 3D voxel field array.

**Data Structures:**
- `field`: A 3D numpy array of shape $(maxZ, maxX, maxY)$ or $(maxX, maxY, maxZ)$, dtype `uint8`. Memory order is row-major (C-style). Access pattern is `field[z][x][y]` [S1, unofficial].
- The field is initialized to 255 (opaque background) [S1, unofficial].

**Parameters:**
- $r$: Sphere radius. Typical values are 1–3 for small bubbles or 7–16 for large ones [S1, unofficial].
- $sep$: Sampling stride for seed points. Typical values are 1, 2, or 10 [S1, unofficial].
- $A, B$: Squared semi-axes for elliptical variants (see Subtopics) [S1, unofficial].

**Steps:**
1. Initialize the 3D `field` array to 255.
2. Sample seed points $(px, py, pz)$ across the grid using a stride $sep$ or random grid traversal [S1, unofficial].
3. For each seed point:
   a. Pick a radius $r$ from the distribution (e.g., `randint(1, 3)`) [S1, unofficial].
   b. Define the bounding box for the sphere: $i \in [px-r-1, px+r+1]$, $j \in [py-r-1, py+r+1]$, $k \in [pz-r-1, pz+r+1]$ [S1, unofficial].
   c. Iterate through every voxel $(i, j, k)$ in the bounding box.
   d. Compute local coordinates relative to the center:
      $$i_2 = i - px$$
      $$j_2 = j - py$$
      $$k_2 = k - pz$$
   e. Perform the sphere inclusion test:
      $$i_2^2 + j_2^2 + k_2^2 < r^2$$
   f. If the test is true, clamp the coordinates to the grid bounds and set the voxel to 0 (transparent):
      $$x = \max(0, \min(i, maxX-1))$$
      $$y = \max(0, \min(j, maxY-1))$$
      $$z = \max(0, \min(k, maxZ-1))$$
      $$field[z][x][y] = 0$$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import random

def procedural_sphere_placement(maxX, maxY, maxZ, sep=2, min_r=1, max_r=3):
    # Initialize field to 255 (opaque)
    field = np.full((maxZ, maxX, maxY), 255, dtype=np.uint8)
    
    # Sample seed points
    # Using a simple grid traversal with stride sep
    for pz in range(0, maxZ, sep):
        for px in range(0, maxX, sep):
            for py in range(0, maxY, sep):
                # Pick random radius
                r = random.randint(min_r, max_r)
                
                # Define bounding box
                i_start = max(0, px - r - 1)
                i_end = min(maxX, px + r + 2)
                j_start = max(0, py - r - 1)
                j_end = min(maxY, py + r + 2)
                k_start = max(0, pz - r - 1)
                k_end = min(maxZ, pz + r + 2)
                
                # Iterate bounding box
                for i in range(i_start, i_end):
                    i2 = i - px
                    i2_sq = i2 * i2
                    for j in range(j_start, j_end):
                        j2 = j - py
                        j2_sq = j2 * j2
                        for k in range(k_start, k_end):
                            k2 = k - pz
                            
                            # Sphere inclusion test
                            if i2_sq + j2_sq + k2_sq < r * r:
                                # Clamp and set to 0
                                field[k][i][j] = 0
                                
    return field
```

**Edge Cases and Pitfalls:**
- Boundary clamping is mandatory to prevent out-of-bounds array access when spheres are placed near grid edges [S1, unofficial].
- Precision issues are minimal due to integer arithmetic for the bounding box and squared distance test, but floating-point radius distributions would require careful handling [G].

## Subtopics

### Elliptical Cylinder Cropping
Removes voxels outside a specified elliptical cylinder boundary, effectively cropping the procedural volume to a specific shape [S1, unofficial]. It differs from standard sphere placement by defining a cylindrical void along one axis rather than spherical voids. Choose this when a non-spherical, elongated void is required.

**Definition:** Iterates over all voxels $(i, j, k)$ and clears those satisfying the elliptical condition [S1, unofficial].

**Steps:**
1. Iterate over all voxels $(i, j, k)$ in the grid [S1, unofficial].
2. Compute centered coordinates with jitter:
   $$i_2 = i - maxX/2.0 + \text{randint}(0, 3)$$
   $$j_2 = j - maxY/2.0 + \text{randint}(0, 3)$$
   $$k_2 = k - maxZ/2.0 + \text{randint}(0, 3)$$ [S1, unofficial].
3. Evaluate the elliptical condition:
   $$\frac{j_2^2}{A} + \frac{k_2^2}{B} > 1$$
   If true, set `field[i][j][k] = 0` [S1, unofficial].

**Parameters:**
- $A, B$: Squared semi-axes of the elliptical cylinder, typically 3000, 3500, or 4500 [S1, unofficial].
- $\text{randint}(0, 3)$: Adds slight jitter to the center offset [S1, unofficial].

**Pitfalls:**
- The `randint(0, 3)` jitter shifts the cylinder center per-voxel, creating a noisy, irregular boundary rather than a smooth mathematical cylinder [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot R^2)$ where $N$ is the number of seed points and $R$ is the average radius, as each seed iterates over its bounding box. With stride $sep$, $N \approx \frac{maxX \cdot maxY \cdot maxZ}{sep^3}$.
- **Memory Complexity:** $O(maxX \cdot maxY \cdot maxZ)$ for the `field` array [S1, unofficial].
- **Strengths:** Simple to implement; produces organic, porous structures suitable for terrain or foam [S1, unofficial].
- **Limitations:** Hard-edged voids may appear unnatural for some materials; jitter in variants can cause visual artifacts [S1, unofficial].
- **When to use:** Destructible terrain, porous materials, or procedural noise generation where spherical voids are desired [S1, unofficial].
- **When not to use:** When smooth density gradients or precise geometric boundaries are required [S1, unofficial].

## Background
Originating from the `webgl-volumetric` project by rbaravalle, this method is used for generating volumetric data with procedural spherical voids [S1, unofficial]. It is applied in WebGL-based volume rendering to create complex internal structures without explicit mesh data [S1, unofficial].

## Sources
[S1] rbaravalle/webgl-volumetric, https://github.com/rbaravalle/webgl-volumetric, fetched 2026-10-27 (unofficial)
