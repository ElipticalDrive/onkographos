# Worley Noise

Also called: F1 Noise, Voronoi Noise, Cellular Noise, Cellular Worley noise, Voronoi F1 distance noise, Voronoï F1 distance noise

## Overview
Worley Noise is a procedural noise generation technique that creates cellular patterns by computing the distance from a query point to the nearest feature point in a grid of pseudo-randomly placed points [S1, S2, S4, S7, unofficial]. It solves the problem of generating structured, organic-looking density variations for volumetric data, such as cloud boundaries or smoke textures, where gradient-based noise (like Perlin) produces smooth gradients rather than distinct cell-like structures [S1, S7, unofficial]. Unlike [Gradient-based Noise (Perlin/Simplex)](3860-gradient-based-noise-perlin-simplex.md) which interpolates values from lattice corners, Worley noise relies on spatial proximity to random centers, making it distinct from [Value Noise](4008-value-noise.md) which interpolates scalar values at lattice points [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\mathbf{P} = (x, y, z)$ in 3D space; a cell size (grid resolution) defining the lattice; a hash function to map integer cell coordinates to random feature points; a distance metric (Euclidean, Manhattan, or Chebyshev); and a combination method for multiple nearest neighbors.
*   **Output:** A scalar value representing the distance to the nearest feature point (F1), or a combination of distances (e.g., F2-F1).

**Data Structures:**
*   **Feature Points:** In GPU implementations, these are often stored in a `ComputeBuffer` (Structured Buffer) with stride `sizeof(float) * 3` for 3D coordinates $(rx, ry, rz) \in [0, 1]^3$ [S1, unofficial]. In CPU implementations, they are generated on-the-fly using a hash function [S4, unofficial].
*   **Distance Array:** A structure to store the distances to the $N$ closest feature points, typically initialized to infinity. For F1, only the minimum is needed. For F2 (second nearest), an array of size 2 is used [S2, unofficial].

**Numbered Steps:**
1.  **Normalize Coordinates:** Convert the world position $\mathbf{P}$ to cell space by dividing by the cell size: $\mathbf{u} = \mathbf{P} / \text{cellSize}$ [S5, unofficial].
2.  **Determine Base Cell:** Calculate the integer coordinates of the cell containing the query point: $(ix, iy, iz) = (\lfloor u_x \rfloor, \lfloor u_y \rfloor, \lfloor u_z \rfloor)$ [S4, S5, unofficial].
3.  **Calculate Fractional Part:** Compute the fractional coordinates within the cell: $(fx, fy, fz) = (u_x - ix, u_y - iy, u_z - iz)$ [S4, unofficial].
4.  **Iterate Neighborhood:** Loop over neighboring integer cells $(i, j, k)$ where $i, j, k \in \{-1, 0, 1\}$ (a 3x3x3 neighborhood) [S3, S4, unofficial]. This covers all cells that could contain a feature point closer than those in further cells [S5, unofficial].
5.  **Hash Feature Point:** For each neighbor cell $(ix+i, iy+j, iz+k)$, generate a pseudo-random feature point $\mathbf{F}_{ijk} = (rx, ry, rz)$ using a hash function [S4, unofficial].
    *   A common hash function for 3D integer coordinates $(cx, cy, cz)$ is:
        $$
        \begin{aligned}
        x &= (cx \cdot 127.1 + cy \cdot 311.7 + cz \cdot 74.7) \% 289.0 \\
        x &= \sin(x) \cdot 43758.5453 \\
        rx &= x - \lfloor x \rfloor
        \end{aligned}
        $$
        Repeat for $y$ and $z$ with different coefficients (e.g., $269.5/183.3/246.1$ and $113.5/271.9/124.6$) [S4, unofficial].
6.  **Compute Distance Vector:** Calculate the vector from the query point to the feature point. Since $\mathbf{F}_{ijk}$ is in $[0,1]$ and the cell offset is $(i,j,k)$, the distance vector $\mathbf{d}$ is:
    $$
    \mathbf{d} = (i + rx - fx, \ j + ry - fy, \ k + rz - fz)
    $$
    [S4, unofficial].
7.  **Calculate Distance Metric:** Compute the distance $dist$ using the chosen metric [S2, unofficial]:
    *   **Euclidean:** $dist = \sqrt{d_x^2 + d_y^2 + d_z^2}$
    *   **Manhattan:** $dist = |d_x| + |d_y| + |d_z|$
    *   **Chebyshev:** $dist = \max(|d_x|, |d_y|, |d_z|)$
    *   *Note:* Some implementations use squared Euclidean distance to avoid the square root, but must be consistent with the combination method [S2, unofficial].
8.  **Update Nearest Neighbors:**
    *   For **F1 (Worley Noise):** Track the minimum distance found so far.
    *   For **F2 (Worley Edge Erosion):** Maintain the two smallest distances, $d_0$ (closest) and $d_1$ (second closest). If $dist < d_0$, shift $d_0$ to $d_1$ and update $d_0 = dist$. Else if $dist < d_1$, update $d_1 = dist$ [S4, unofficial].
9.  **Combine and Scale:** Apply the combination method and scale by an amplitude factor [S2, unofficial].
    *   **D0:** $output = d_0$
    *   **D1_D0:** $output = d_1 - d_0$
    *   Normalize the result to $[0, 1]$ if necessary (e.g., divide by $\sqrt{3}$ for Euclidean in a unit cube and clamp) [S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hash_3d(ix, iy, iz):
    """
    Hashes 3D integer coordinates to a random float vector in [0, 1]^3.
    Based on [S4, unofficial].
    """
    # X component
    x = (ix * 127.1 + iy * 311.7 + iz * 74.7) % 289.0
    x = math.sin(x) * 43758.5453
    rx = x - math.floor(x)
    
    # Y component
    y = (ix * 269.5 + iy * 183.3 + iz * 246.1) % 289.0
    y = math.sin(y) * 43758.5453
    ry = y - math.floor(y)
    
    # Z component
    z = (ix * 113.5 + iy * 271.9 + iz * 124.6) % 289.0
    z = math.sin(z) * 43758.5453
    rz = z - math.floor(z)
    
    return (rx, ry, rz)

def worley_noise(x, y, z, cell_size, distance_metric="euclidean", combination="D0"):
    """
    Computes Worley Noise for a given point.
    
    Parameters:
    - x, y, z: Query point coordinates.
    - cell_size: Size of the grid cells.
    - distance_metric: "euclidean", "manhattan", or "chebyshev".
    - combination: "D0" (F1) or "D1_D0" (F2-F1).
    
    Returns:
    - Normalized scalar value.
    """
    # 1. Normalize to cell space
    u = x / cell_size
    v = y / cell_size
    w = z / cell_size
    
    # 2. Base cell and fractional part
    ix = math.floor(u)
    iy = math.floor(v)
    iz = math.floor(w)
    
    fx = u - ix
    fy = v - iy
    fz = w - iz
    
    # Initialize distances
    # For D0, we only need min_dist. For D1_D0, we need d0 and d1.
    d0 = float('inf')
    d1 = float('inf')
    
    # 3. Iterate over 3x3x3 neighborhood
    for i in range(-1, 2):
        for j in range(-1, 2):
            for k in range(-1, 2):
                # 4. Hash feature point for neighbor cell (ix+i, iy+j, iz+k)
                rx, ry, rz = hash_3d(ix + i, iy + j, iz + k)
                
                # 5. Distance vector
                dx = i + rx - fx
                dy = j + ry - fy
                dz = k + rz - fz
                
                # 6. Calculate distance based on metric
                if distance_metric == "euclidean":
                    dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                elif distance_metric == "manhattan":
                    dist = abs(dx) + abs(dy) + abs(dz)
                elif distance_metric == "chebyshev":
                    dist = max(abs(dx), abs(dy), abs(dz))
                else:
                    raise ValueError("Unknown distance metric")
                
                # 7. Update nearest neighbors
                if dist < d0:
                    d1 = d0
                    d0 = dist
                elif dist < d1:
                    d1 = dist
    
    # 8. Combine
    if combination == "D0":
        result = d0
    elif combination == "D1_D0":
        result = d1 - d0
    else:
        raise ValueError("Unknown combination method")
        
    # 9. Normalize (assuming Euclidean in unit cube, max dist is sqrt(3))
    # Note: This normalization is specific to Euclidean distance in [0,1] cells.
    # For other metrics or if raw distance is desired, skip this.
    if distance_metric == "euclidean":
        result = result / math.sqrt(3.0)
        result = max(0.0, min(1.0, result))
        
    return result
```

**Parameters:**
*   **Cell Size:** Controls the frequency of the cellular pattern. Smaller cells create finer detail [S1, unofficial].
*   **Distance Metric:** Euclidean is standard for organic shapes; Manhattan/Chebyshev can create sharper, more geometric patterns [S2, unofficial].
*   **Combination:** "D0" returns raw distance to nearest point (F1). "D1_D0" returns the difference between the second and first nearest distances, highlighting cell edges (F2-F1) [S2, S4, unofficial].
*   **Amplitude:** A multiplier applied to the final output to control contrast [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Distance Calculation Typo:** A common implementation error is using `dz * dy` instead of `dz * dz` in the Euclidean distance formula, leading to incorrect noise [S4, unofficial].
*   **Boundary Conditions:** When iterating neighbors, ensure the hash function is consistent and handles negative indices correctly if the query point is near the origin or boundaries of a bounded volume [S4, unofficial].
*   **Precision:** Using squared Euclidean distance avoids square roots but requires careful handling when combining with other metrics or normalizing [S2, unofficial].

## Subtopics

### [Animated Worley Noise Texture Generation](2982-animated-worley-noise-texture-generation.md)
Generates a 2D Worley noise texture to break up smoke silhouettes, updated per frame via compute shader. It differs from its parent [Worley Noise](1726-worley-noise.md) by explicitly incorporating a time uniform to animate the point positions, and from [Voronoise](5519-voronoise.md) by returning raw distance values rather than smoothed vector displacements [G].

### [Voronoise](5519-voronoise.md)
Voronoise is a procedural noise function that unifies value noise and Voronoi cellular patterns through a single interpolation parameter. It solves the problem of needing distinct noise types for continuous textures and cellular structures by providing a smooth transition between them. Unlike standard Worley noise which only returns distance metrics, Voronoise returns a continuous scalar value suitable for fractal Brownian motion (FBM) accumulation, distinguishing it from its parent [1726-worley-noise.md] and sibling [2982-animated-worley-noise-texture-generation.md].

### Worley Edge Erosion (F2 - F1)
Generates crystalline edge erosion textures for cloud boundary curling by computing the difference between the second nearest neighbor distance (F2) and the nearest neighbor distance (F1) [S4, unofficial]. It is implemented by tracking the two closest distances ($d_0, d_1$) in the 3x3x3 neighborhood and returning $\max(0.0, \min(1.0, d_1 - d_0))$ [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, as it iterates over a fixed-size neighborhood (27 cells in 3D) [S3, S4, unofficial].
*   **Memory Complexity:** $O(1)$ if feature points are hashed on-the-fly, or $O(N)$ if pre-computed feature points are stored in a buffer, where $N$ is the number of cells in the volume [S1, unofficial].
*   **Performance:** GPU implementations using compute shaders can generate noise textures efficiently by dispatching thread groups corresponding to the output resolution [S1, unofficial]. The hash function is lightweight, making it suitable for real-time applications [S7, unofficial].
*   **Strengths:** Produces distinct, organic cellular patterns; easy to animate by shifting feature points; computationally inexpensive [S7, unofficial].
*   **Limitations:** The pattern is periodic if the hash function or cell grid is not handled carefully; lacks the smooth, directional flow of gradient noise [S2, unofficial].
*   **When to Use:** Ideal for generating cloud bases, smoke textures, or any volumetric data requiring a "cellular" or "bubbly" appearance [S1, S7, unofficial].
*   **When Not to Use:** When smooth, continuous gradients or directional flow are required (use [Gradient-based Noise (Perlin/Simplex)](3860-gradient-based-noise-perlin-simplex.md) instead) [S2, unofficial].

## Background
Worley Noise was introduced by Steven Worley in 1996 in the paper "A Cellular Texture Basis Function" [S5, S7, unofficial]. It is based on Voronoi diagrams, where the space is partitioned into regions based on distance to a set of random points [S2, S7, unofficial]. The method has been widely adopted in computer graphics for procedural texture generation, particularly in volumetric rendering for clouds and smoke, due to its ability to create realistic, organic-looking cellular structures [S1, S7, unofficial].

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
[S2] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S3] M4trixStrike/WebGL-Volumetric-clouds, https://github.com/M4trixStrike/WebGL-Volumetric-clouds, fetched 2026-10-27 (unofficial)
[S4] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
[S5] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S6] Schokolasbotter/volumetric-smoke-raymarching, https://github.com/Schokolasbotter/volumetric-smoke-raymarching, fetched 2026-10-28 (unofficial)
[S7] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
