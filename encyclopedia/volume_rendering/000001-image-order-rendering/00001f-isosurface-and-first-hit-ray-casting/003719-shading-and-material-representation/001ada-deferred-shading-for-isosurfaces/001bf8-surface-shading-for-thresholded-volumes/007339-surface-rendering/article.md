# Surface Rendering
Also called: Surface-oriented rendering

## Overview
Surface rendering is a visualization technique that extracts the outer boundary of a segmented structure and constructs a geometric shell, typically composed of triangles, to represent the object as a solid surface [S1, unofficial]. Unlike volume rendering, which maps every voxel to color and transparency to create a see-through block, surface rendering treats the object as having a uniform surface color where shading indicates light source location [S2]. This method is primarily used for visualizing the shape of bones, fracture fragments, organ exteriors, and printable models, offering faster rendering speeds because only surface vertices need manipulation rather than every voxel in the volume [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A segmented 3D volume (binary mask or labeled regions) derived from medical imaging (e.g., CT or MRI) [S1, unofficial].
*   **Output:** A triangular mesh (surface shell) representing the outer boundary of the segmented structure [S1, unofficial].

**Data Structures:**
*   **Mesh:** A collection of vertices and faces.
    *   **Vertices:** 3D coordinates $(x, y, z)$ corresponding to the extracted boundary points.
    *   **Faces:** Indices referencing vertices to form triangles.
    *   **Attributes:** Surface color (uniform), normals (for shading), and optionally texture coordinates [S3].

**Steps:**
1.  **Boundary Extraction:** Identify the interface between foreground (1) and background (0) voxels [S3].
2.  **Mesh Generation:** Build a shell of triangles connecting the boundary points to create a solid object representation [S1, unofficial].
3.  **Shading Application:** Apply lighting models to the surface to convey the location of light sources and surface orientation [S2].
4.  **Rendering:** Render the triangular mesh using standard rasterization techniques, typically with perspective projection to create a depth illusion [S2, S3].

**Formulas:**
Surface normals $\mathbf{n}$ for each triangle are computed to determine lighting interaction:
$$\mathbf{n} = \frac{(\mathbf{v}_2 - \mathbf{v}_1) \times (\mathbf{v}_3 - \mathbf{v}_1)}{\|(\mathbf{v}_2 - \mathbf{v}_1) \times (\mathbf{v}_3 - \mathbf{v}_1)\|}$$
where $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$ are the triangle vertices and $\times$ denotes the cross product [G].

**Parameters and Edge Cases:**
*   **Thresholding:** To achieve sharp edges, high air/surface thresholds are often required, but this can strip gray matter from the image, leading to an inaccurate representation of brain size [S2].
*   **Erosion:** Gray matter is typically eroded to improve edge definition, which further contributes to size inaccuracy [S2].
*   **Pitfalls:** Low-resolution images or average contrast clinical images cause problems with clean edge extraction [S2]. The surface can hide deeper internal damage or structures, making it misleading if used in isolation [S2]. Surface color does not reflect underlying tissue unless texture maps are used [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_render(volume):
    vertices = []
    faces = []
    face_cache = {}
    # Exact mesh extraction algorithm not specified in sources [G]
    for z in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[2]):
                if volume[z, y, x] == 0:
                    continue
                for dz, dy, dx in [(-1,0,0), (1,0,0), (0,-1,0), (0,1,0), (0,0,-1), (0,0,1)]:
                    nz, ny, nx = z + dz, y + dy, x + dx
                    if 0 <= nz < volume.shape[0] and 0 <= ny < volume.shape[1] and 0 <= nx < volume.shape[2]:
                        if volume[nz, ny, nx] == 0:
                            # Generate quad for the boundary face between (x,y,z) and (nx,ny,nz)
                            # Coordinates are in (x, y, z) space
                            if dz == -1: corners = [(x,y,z), (x+1,y,z), (x,y+1,z), (x+1,y+1,z)]
                            elif dz == 1: corners = [(x,y,z+1), (x+1,y,z+1), (x,y+1,z+1), (x+1,y+1,z+1)]
                            elif dy == -1: corners = [(x,y,z), (x+1,y,z), (x,y,z+1), (x+1,y,z+1)]
                            elif dy == 1: corners = [(x,y+1,z), (x+1,y+1,z), (x,y+1,z+1), (x+1,y+1,z+1)]
                            elif dx == -1: corners = [(x,y,z), (x,y+1,z), (x,y,z+1), (x,y+1,z+1)]
                            else: corners = [(x+1,y,z), (x+1,y+1,z), (x+1,y,z+1), (x+1,y+1,z+1)]
                            key = tuple(corners)
                            if key not in face_cache:
                                v0 = len(vertices)
                                for c in corners:
                                    vertices.append(c)
                                face_cache[key] = v0
                            else:
                                v0 = face_cache[key]
                            faces.append((v0, v0+1, v0+2))
                            faces.append((v0, v0+2, v0+3))
    normals = compute_normals(vertices, faces)
    return vertices, faces, normals

def compute_normals(vertices, faces):
    normals = []
    for v0, v1, v2 in faces:
        p0 = vertices[v0]
        p1 = vertices[v1]
        p2 = vertices[v2]
        u = (p1[0]-p0[0], p1[1]-p0[1], p1[2]-p0[2])
        v = (p2[0]-p0[0], p2[1]-p0[1], p2[2]-p0[2])
        cross = (u[1]*v[2]-u[2]*v[1], u[2]*v[0]-u[0]*v[2], u[0]*v[1]-u[1]*v[0])
        mag = (cross[0]**2 + cross[1]**2 + cross[2]**2)**0.5
        if mag > 0:
            normals.append((cross[0]/mag, cross[1]/mag, cross[2]/mag))
        else:
            normals.append((0.0, 0.0, 1.0))
    return normals
```

## Operating characteristics
*   **Time Complexity:** Generally faster than volume rendering because only surface vertices need to be manipulated and recomputed following rotations, whereas volume rendering requires recomputing every voxel [S2].
*   **Memory Complexity:** Depends on the resolution of the extracted mesh [G].
*   **Strengths:** Produces a solid object suitable for measurements, surgical planning, and 3D printing. It is fast and provides clear shape visualization [S1, unofficial].
*   **Limitations:** Requires high-quality scans and excellent segmentation to avoid artifacts. Surface color does not reflect underlying tissue unless texture maps are used [S2]. It can hide deeper internal structures and may produce inaccurate size representations due to erosion or thresholding [S2].
*   **When to Use:** For visualizing the shape of bones, fracture fragments, organ exteriors, or when creating printable models [S1, unofficial].
*   **When Not to Use:** When internal structures, tissue intensity variations, or hidden damage need to be visualized [S2].

## Background
Surface rendering is widely used in oncology for tumor and organ volume measurements, orthopedics for fracture and implant planning, and interventional radiology for vessel mapping [S1, unofficial]. Tools like VolumeJ are examples of surface renderers that utilize shading to indicate light source location [S2]. The technique is often used in conjunction with stereotactically aligned slices to mitigate its limitations in showing internal structures [S2].

## Subtopics

### Shading for Isosurfaces (Single-Pass Multi-Stage)
Focuses on efficient single-pass shading techniques for isosurfaces, differing from standard surface rendering by optimizing the shading pipeline for isosurface geometry specifically. Choose when rendering performance on isosurfaces is critical. [Shading for Isosurfaces (Single-Pass Multi-Stage)](2094-shading-for-isosurfaces-single-pass-multi-stage.md)

### Fast Shaded Isosurfaces
Optimizes the rendering speed of shaded isosurfaces, offering a performance advantage over general surface rendering methods in specific hardware configurations. Choose when hardware acceleration for isosurface shading is available. [Fast Shaded Isosurfaces](2105-fast-shaded-isosurfaces.md)

### Volume Raycasting with Surface Extraction and Phong Shading
Combines volume raycasting with surface extraction, allowing for Phong shading on the extracted surface while retaining some volume properties. Choose when a hybrid approach between volume and surface rendering is desired. [Volume Raycasting with Surface Extraction and Phong Shading](3662-volume-raycasting-with-surface-extraction-and-phong-shading.md)

### Interval Shading
Uses interval arithmetic for shading, providing robustness against numerical errors in surface representation, unlike standard surface rendering which may suffer from precision issues. Choose when numerical stability in shading calculations is required. [Interval Shading](6414-interval-shading.md)

### IsosurfaceClassifier Lighting Model
Applies a specialized lighting model for isosurfaces that considers classification data, differing from the uniform color approach of basic surface rendering. Choose when classification data influences surface appearance. [IsosurfaceClassifier Lighting Model](6676-isosurfaceclassifier-lighting-model.md)

### Gaussian Grounded Inter-Reflection
Models inter-reflections on surfaces using Gaussian distributions, offering more realistic lighting than the simple shading used in standard surface rendering. Choose when realistic global illumination effects on surfaces are needed. [Gaussian Grounded Inter-Reflection](13228-gaussian-grounded-inter-reflection.md)

### Boundary Shader
Focuses on shading specifically at the boundaries of volumes, potentially offering more detail at edges than the uniform surface shading of standard surface rendering. Choose when edge detail and boundary-specific lighting are prioritized. [Boundary Shader](14912-boundary-shader.md)

## Sources
[S1] 3D reconstruction from CT, https://pycad.co/blog/3-d-reconstruction-ct/, fetched 2026-10-26 (unofficial)
[S2] Volume Rendering the Cortex - University of South Carolina, https://people.cas.sc.edu/rorden/mricro/render/index.html, fetched 2026-10-27
[S3] 4. Displaying data — ParaView Documentation 5.8.1 documentation, https://docs.paraview.org/en/v5.9.1/UsersGuide/displayingData.html, fetched 2026-10-29
