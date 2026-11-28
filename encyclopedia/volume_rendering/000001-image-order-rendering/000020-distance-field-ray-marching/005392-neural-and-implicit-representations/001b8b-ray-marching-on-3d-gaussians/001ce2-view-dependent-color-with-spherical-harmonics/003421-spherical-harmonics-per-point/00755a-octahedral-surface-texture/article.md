# Octahedral Surface Texture

Also called: Paired Octahedral Surface and View-Dependent Textures, Octahedral Appearance Textures

## Overview
Octahedral Surface Texture is a per-cell appearance representation that replaces spherical harmonics with an $8 \times 8$ RGB texture projected onto a Voronoi cell's surface [S1, unofficial]. It solves the problem of limited spatial appearance variation within a single primitive by allowing color to vary across the cell's image footprint, enabling larger cells to represent spatial detail that would otherwise require additional primitives [S1, unofficial]. It sets itself apart from its parent [Spherical Harmonics Per Point](7172-spherical-harmonics-per-point.md) by halving the appearance values required for each cell evaluation while providing continuous, singularity-free directional mapping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The hit point $\mathbf{x}$ of a ray intersecting a Voronoi cell $V_i$, the site position $\mathbf{p}_i$ of that cell, and the per-cell texture $T_i$ (an $8 \times 8$ RGB texture).
*   **Output:** The sampled color $\mathbf{c}$ at the hit point.

**Data Structures:**
*   **Texture $T_i$:** An $8 \times 8$ grid of RGB values. Each texel stores 3 floating-point components. Total size: $64 \times 3$ floats per cell.

**Steps:**
1.  **Compute Direction:** Calculate the unit direction vector $\mathbf{d}$ from the site to the hit point:
    $$ \mathbf{d} = \frac{\mathbf{x} - \mathbf{p}_i}{\|\mathbf{x} - \mathbf{p}_i\|} $$
2.  **Project to Octahedron:** Project $\mathbf{d}$ onto the unit octahedron using $\ell_1$ normalization to obtain coordinates $\mathbf{q}$:
    $$ \mathbf{q} = \frac{\mathbf{d}}{|d_x| + |d_y| + |d_z|} $$
3.  **Map to Texture Coordinates:** Convert $\mathbf{q}$ to 2D texture coordinates $(u, v) \in [-1, 1]^2$ based on the sign of $d_z$:
    $$ (u, v) = \begin{cases} (q_x, q_y) & \text{if } d_z \geq 0 \\ ((1 - |q_y|) \operatorname{sign}(q_x), (1 - |q_x|) \operatorname{sign}(q_y)) & \text{if } d_z < 0 \end{cases} $$
    *   If $d_z \geq 0$ (upper hemisphere), the mapping is direct.
    *   If $d_z < 0$ (lower hemisphere), the mapping unfolds the lower face to the corners of the square domain to maintain continuity across the texture boundary [S1, unofficial].
4.  **Sample Texture:** Rescale $(u, v)$ from $[-1, 1]^2$ to the texel space of $T_i$ (e.g., $[0, 7]$ or normalized $[0, 1]$ depending on implementation) and perform bilinear interpolation to retrieve the color $\mathbf{c}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def octahedral_surface_texture(hit_point: Vector3, site: Vector3, texture: Texture8x8) -> Vector3:
    # 1. Compute unit direction
    direction = hit_point - site
    d = normalize(direction)
    
    # 2. Project to unit octahedron (l1 normalization)
    l1_norm = abs(d.x) + abs(d.y) + abs(d.z)
    q = Vector3(d.x / l1_norm, d.y / l1_norm, d.z / l1_norm)
    
    # 3. Map to texture coordinates (u, v) in [-1, 1]
    if d.z >= 0:
        u = q.x
        v = q.y
    else:
        u = (1 - abs(q.y)) * sign(q.x)
        v = (1 - abs(q.x)) * sign(q.y)
        
    # 4. Sample texture
    # Rescale u, v from [-1, 1] to [0, 1] for standard texture lookup
    u_norm = (u + 1.0) / 2.0
    v_norm = (v + 1.0) / 2.0
    
    # Bilinear interpolation on the 8x8 RGB texture
    color = texture.sample_bilinear(u_norm, v_norm)
    
    return color
```

**Parameters:**
*   **Texture Size:** $8 \times 8$ RGB [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Direction:** If $\mathbf{x} = \mathbf{p}_i$, the direction is undefined. In practice, this occurs only if the ray origin is exactly at the site, which is rare in ray marching. A guard should handle division by zero in normalization [G].
*   **Boundary Continuity:** The mapping is designed to be continuous across the texture boundary. Incorrect handling of the $\operatorname{sign}(0)$ case in the lower hemisphere mapping can cause artifacts. $\operatorname{sign}(0)$ is typically defined as 0 or 1 depending on convention, but since $q_x, q_y$ are non-zero at the exact center of the lower face (where $d_z < 0$ and $q_x=q_y=0$ is impossible for a valid direction on the octahedron surface unless $d_z=0$), this is generally stable [S1, unofficial].

## Subtopics

### Octahedral Appearance Textures
A technique to store view-dependent color using octahedral maps to add spatial and directional detail while keeping appearance traffic bounded, replacing spherical harmonics [S1, unofficial]. It is chosen when the primary goal is to reduce per-cell appearance cost while maintaining spatial variation within a cell, as opposed to using spherical harmonics which require more coefficients for similar directional accuracy [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per cell evaluation. The operations involve a few vector arithmetic operations and a texture lookup.
*   **Memory Complexity:** $O(1)$ per cell. Each cell stores a fixed $8 \times 8$ RGB texture ($64 \times 3$ floats). This is significantly less than spherical harmonics, which require more coefficients for comparable directional accuracy [S1, unofficial].
*   **Performance:** The method reduces appearance traffic and per-cell evaluation cost compared to spherical harmonics [S1, unofficial]. It allows for larger cells, reducing the total number of primitives needed to represent spatial detail [S1, unofficial].
*   **Strengths:** Low distortion, no polar singularities, continuous across texture boundaries, reduced memory footprint compared to SH [S1, unofficial].
*   **Limitations:** The $8 \times 8$ resolution may not capture high-frequency spatial details as well as denser primitives or higher-resolution textures. It is limited to the specific Voronoi cell structure where the site is interior and the map from direction to surface is a bijection [S1, unofficial].
*   **When to use:** When rendering with a fixed budget of primitives and needing to balance spatial and directional appearance detail with memory constraints [S1, unofficial].
*   **When not to use:** When extremely high-frequency spatial detail is required within a single cell, or when the primitive structure does not allow for a bijection between direction and surface point [S1, unofficial].

## Background
The method was introduced in "Differentiable Voronoi Ray Tracing Beyond Rasterization Speeds" [S1, unofficial]. It builds on the concept of textured Gaussians [Chao et al. 2025] but adapts it for Voronoi-based rendering where appearance lookup is part of the per-cell traversal loop. The octahedral mapping technique itself is based on [Cigolle et al. 2014] [S1, unofficial]. The approach is part of a broader effort to optimize explicit radiance-field methods by reducing per-cell appearance cost and enabling larger cells to represent spatial detail [S1, unofficial].

## Sources
[S1] Differentiable Voronoi Ray Tracing Beyond Rasterization Speeds, https://arxiv.org/abs/2608.17682, fetched 2026-10-27 (unofficial)
