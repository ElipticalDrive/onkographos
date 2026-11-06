# Gouraud Shading

Also called: intensity interpolation shading, smooth shading

## Overview
Gouraud shading is a vertex-based illumination technique that computes lighting colors at mesh vertices and linearly interpolates these values across the resulting primitives to simulate smooth surface shading [S1]. It differs from sibling methods by interpolating the final color rather than the geometric normal vector; [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) interpolates normals to re-evaluate the lighting model per-pixel, while [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) and [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md) apply specific diffuse or specular models at the sample or pixel level.

## How it works
**Inputs:**
- A mesh defined by vertices $V_i$ and faces $F_j$.
- For each vertex $V_i$, a normal vector $\mathbf{n}_i$ and position $\mathbf{p}_i$.
- A light source direction $\mathbf{l}$ (unit vector pointing from surface to light).
- A surface albedo or base color $C_{base}$.
- A diffuse lighting model (typically Lambertian).

**Outputs:**
- A rasterized image with interpolated colors across primitives.

**Data Structures:**
- Vertex structure: $\{ \mathbf{p}_i, \mathbf{n}_i, C_i \}$, where $C_i$ is the computed vertex color.
- Primitive structure: list of vertex indices.

**Steps:**
1. **Compute Vertex Normals:** For each vertex $V_i$, determine the normal vector $\mathbf{n}_i$.
   - For planar surfaces, assign the same normal to all vertices of the face.
   - For curved surfaces, compute $\mathbf{n}_i$ as the normalized average of the normals of all faces $F_j$ sharing vertex $V_i$ [S2, unofficial].
   $$ \mathbf{n}_i = \frac{\sum_{F_j \ni V_i} \mathbf{n}_{F_j}}{\left\| \sum_{F_j \ni V_i} \mathbf{n}_{F_j} \right\|} $$
   - Failure to average normals for curved surfaces results in discontinuous slopes and a faceted appearance [S2, unofficial].

2. **Compute Vertex Colors:** For each vertex $V_i$, calculate the color $C_i$ using the lighting model.
   - Calculate the diffuse term using the dot product of the vertex normal $\mathbf{n}_i$ and the light direction $\mathbf{l}$ [S2, unofficial].
   $$ C_i = C_{base} \cdot \max(0, \mathbf{n}_i \cdot \mathbf{l}) $$
   - Note: The sources describe the angle between the light source direction and the vertex normal vector as the determining factor for color [S2, unofficial].

3. **Interpolate Colors:** For each primitive (triangle), linearly interpolate the vertex colors $C_i$ across the surface to determine the color of each fragment/pixel [S1].
   - If barycentric coordinates are $(\alpha, \beta, \gamma)$ for vertices $V_1, V_2, V_3$:
   $$ C_{fragment} = \alpha C_1 + \beta C_2 + \gamma C_3 $$

**Parameters:**
- Normal vector: A vertex attribute perpendicular to the surface, used to estimate light reflection [S2, unofficial].

**Pitfalls:**
- **Faceting:** If normals are not averaged for curved surfaces, the discontinuity in slope between adjacent faces results in a faceted or gem-like appearance [S2, unofficial].
- **Highlight Artifacts:** Because highlights are computed at vertices and interpolated, sharp specular highlights may be lost or distorted if they fall within the interior of a triangle rather than near a vertex [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gouraud_shading(mesh, light_direction, base_color):
    # Step 1: Compute averaged vertex normals
    vertex_normals = {}
    for face in mesh.faces:
        # Compute face normal from vertices
        v0, v1, v2 = face.vertices
        n_face = normalize(cross(v1 - v0, v2 - v0))
        # Accumulate normal for each vertex
        for v in [v0, v1, v2]:
            if v not in vertex_normals:
                vertex_normals[v] = vec3(0, 0, 0)
            vertex_normals[v] += n_face
    
    # Normalize averaged normals
    for v in vertex_normals:
        vertex_normals[v] = normalize(vertex_normals[v])

    # Step 2: Compute vertex colors
    vertex_colors = {}
    for v, n in vertex_normals.items():
        # Lambertian diffuse approximation
        diff = max(0, dot(n, light_direction))
        vertex_colors[v] = base_color * diff

    # Step 3: Interpolate colors across primitives
    for face in mesh.faces:
        c0 = vertex_colors[face.vertices[0]]
        c1 = vertex_colors[face.vertices[1]]
        c2 = vertex_colors[face.vertices[2]]
        
        # Rasterize triangle with vertex colors c0, c1, c2
        rasterize_triangle(face.vertices, c0, c1, c2)
```

## Subtopics

### [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md)
Interpolates normals per-pixel to re-evaluate lighting; choose when sharp specular highlights are required.

### [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md)
Evaluates diffuse lighting directly on the SDF gradient; choose for ray-marched implicit surfaces without explicit meshes.

### [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md)
Uses half-vector for specular highlights in volume rendering; choose when glossy effects are needed in volumetric data.

## Operating characteristics
- **Time Complexity:** $O(V)$ for normal computation and color calculation, where $V$ is the number of vertices. Rasterization is $O(P)$ where $P$ is pixels.
- **Memory Complexity:** $O(V)$ to store vertex colors and normals.
- **Strengths:** Computationally inexpensive; reduces per-pixel lighting calculations to per-vertex calculations [S1].
- **Limitations:** Cannot accurately represent specular highlights that occur away from vertices; may exhibit banding artifacts if color interpolation is not performed in a linear color space [S1].
- **When to use:** Real-time rendering of diffuse surfaces where performance is critical and specular highlights are not prominent [S2, unofficial].

## Background
Gouraud shading, also known as intensity interpolation shading, was introduced as a method to create smoother representations of surfaces by computing color variations within a facet based on surface normals [S1]. It is implemented in OpenGL via `glShadeModel(GL_SMOOTH)` [S2, unofficial]. The technique is used in applications such as Smokeview for visualizing fire dynamics simulation data, where it provides smooth lighting for blockages and iso-surfaces [S2, unofficial].

## Sources
[S1] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
