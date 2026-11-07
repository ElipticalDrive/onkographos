# Texture Projection


## Overview
Texture projection maps 2D textures onto geometric and volumetric data by projecting textures through space onto surfaces [S1, unofficial]. It solves the difficulty of wrapping textures around volumetrically defined objects, which requires knowing where their defining surfaces lie, by projecting through space instead of relying on explicit surface parameterization [S1, unofficial]. It is handled as a straightforward extension of the shadow casting algorithm [S1, unofficial].

## How it works
**Inputs:**
*   Volumetric datasets [S1, unofficial].
*   Geometric polygons (optional, embedded in volumetric data) [S1, unofficial].
*   2D texture maps [S1, unofficial].

**Outputs:**
*   Textured surface appearance for the volumetric object or embedded polygons [S1, unofficial].

**Data Structures:**
*   Texture coordinates $(u, v)$ generated per sample point on the surface [G].
*   Sample point position $\mathbf{p}$ in world or view space [G].

**Steps:**
1. **Determine Surface Location:** Identify the location of the defining surface of the volumetric object [S1, unofficial].
2. **Project Texture Coordinates:** For each point $\mathbf{p}$ on the surface, compute the corresponding 2D texture coordinates $(u, v)$ by projecting $\mathbf{p}$ through the texture space [G].
3. **Sample Texture:** Retrieve the color value from the 2D texture map using the computed $(u, v)$ coordinates [G].
4. **Composite:** Blend the sampled texture color with the underlying volumetric or geometric shading [G].

**Formulas:**
Let $\mathbf{p} = (x, y, z)$ be the 3D point on the surface. The projection to 2D texture coordinates $(u, v)$ uses a projective transformation matrix $M$:
$$
\begin{bmatrix} u' \\ v' \\ w' \end{bmatrix} = M \begin{bmatrix} x \\ y \\ z \\ 1 \end{bmatrix}
$$
$$
u = \frac{u'}{w'}, \quad v = \frac{v'}{w'}
$$
Where $(u, v)$ are normalized to $[0, 1]$ [G].

**Parameters:**
*   Projection Matrix $M$: Defines the view frustum or orthographic projection for the texture [G].
*   Texture Resolution: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Surface Ambiguity:** Wrapping textures around volumetrically defined objects is difficult because it requires knowing where their defining surfaces lie [S1, unofficial].
*   **Spectral Mismatch:** Geometric, volumetric, and texture data each exhibit different spectra, requiring careful handling when mixing them [S1, unofficial].
*   **Embedded Polygons:** Mapping textures onto polygons embedded in volumetric datasets is relatively simple [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_projection(surface_points, texture_map, projection_matrix):
    """Projects a 2D texture onto a set of 3D surface points."""
    colors = []
    rows = len(texture_map)
    cols = len(texture_map[0])
    for point in surface_points:
        x, y, z = point
        # Project to texture space using 3x4 projection matrix
        u_prime = projection_matrix[0][0]*x + projection_matrix[0][1]*y + projection_matrix[0][2]*z + projection_matrix[0][3]
        v_prime = projection_matrix[1][0]*x + projection_matrix[1][1]*y + projection_matrix[1][2]*z + projection_matrix[1][3]
        w_prime = projection_matrix[2][0]*x + projection_matrix[2][1]*y + projection_matrix[2][2]*z + projection_matrix[2][3]
        
        # Perspective divide
        if w_prime == 0.0:
            colors.append(None)
            continue
            
        u = u_prime / w_prime
        v = v_prime / w_prime
        
        # Check bounds and sample texture
        if 0.0 <= u <= 1.0 and 0.0 <= v <= 1.0:
            u_idx = int(u * (cols - 1))
            v_idx = int(v * (rows - 1))
            colors.append(texture_map[v_idx][u_idx])
        else:
            colors.append(None)
    return colors
```

## Subtopics

### [Triplanar Texture Projection](2303-triplanar-texture-projection.md)
Triplanar texture projection maps a 2D texture onto a 3D surface by applying planar mapping in three perpendicular planes (XY, XZ, YZ) and interpolating the results based on the surface normal's alignment. It solves the problem of texture stretching that occurs with single-axis planar mapping on complex or curved surfaces, such as spheres, by ensuring the texture orientation adapts to the local geometry. Unlike its siblings, it avoids visible seams and stretching by weighting three orthogonal projections, making it suitable for shading complex volumetric surfaces. It builds on [Texture Projection](2002-texture-projection.md).

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Easier to implement than surface wrapping for volumetric data; integrates with shadow casting algorithms [S1, unofficial].
*   **Limitations:** Requires determining surface location for volumetric objects; potential artifacts from spectral mismatch between data types [S1, unofficial].
*   **When to use:** When rendering mixtures of geometric and volumetric data where explicit surface parameterization is difficult or unavailable [S1, unofficial].

## Background
Texture projection was introduced in the context of rendering mixtures of geometric and volumetric data [S1, unofficial]. The approach is similar to that described by Feibush, Levoy, and Cook [1980] [S1, unofficial]. A good survey of texture mapping algorithms was written by Heckbert [1986] [S1, unofficial]. The technique is part of the broader field of volume rendering, which encompasses both opaque voxel overpainting and semi-transparent voxel blending [S1, unofficial].

## Sources
[S1] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
[S2] Triplanar Texture Projection, https://example.com/triplanar, fetched 2026-10-26 (unofficial)
