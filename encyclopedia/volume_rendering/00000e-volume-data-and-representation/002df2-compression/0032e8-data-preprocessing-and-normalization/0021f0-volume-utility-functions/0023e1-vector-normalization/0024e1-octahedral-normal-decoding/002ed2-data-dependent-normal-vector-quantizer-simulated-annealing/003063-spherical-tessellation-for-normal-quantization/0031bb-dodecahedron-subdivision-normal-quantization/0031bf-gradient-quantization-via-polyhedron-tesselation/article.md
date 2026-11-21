# Gradient Quantization via Polyhedron Tesselation

Also called: Gradient Quantization via Icosahedron-Dodecahedron Tesselation, Icosahedron-Dodecahedron Gradient Quantization

## Overview
Gradient Quantization via Polyhedron Tesselation quantizes the space of gradient directions into a finite set of unit vectors by tessellating a unit sphere with triangles derived from an icosahedron and its dual, the dodecahedron [S1, unofficial][S2, unofficial]. It solves the problem of uneven directional distribution in precomputed gradient storage, which causes artifacts in directional lighting effects during volume rendering [S1, unofficial][S2, unofficial]. It sets itself apart from octahedron-based subdivision by using triangles that are more nearly planar, thereby undergoing less distortion during recursive subdivision and producing a more uniform distribution of gradient directions [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: The number of refinement levels $k$ (integer $\ge 0$) [S1, unofficial].
- Output: A set of $g$ unit vectors representing quantized gradient directions, stored as vertices on the unit sphere [S2, unofficial].

**Data Structures:**
The method generates a list of 3D coordinates [S2, unofficial]. Each coordinate is a normalized 3-tuple $(x, y, z)$ representing a vertex on the unit sphere [G]. The total number of vertices $g$ is determined by the formula below [S1, unofficial].

**Steps:**
1. **Initialize Base Polyhedra:** Identify the 12 vertices of a regular icosahedron inscribed in the unit sphere and the 20 vertices of a regular dodecahedron inscribed in the unit sphere [G]. Combine these sets to form a pool of 32 unique vertices [S1, unofficial]. Note that the icosahedron and dodecahedron are duals; their vertices correspond to the faces and vertices of each other, respectively [G].
2. **Form Initial Triangles:** Connect the 32 vertices to form 60 congruent (but not equilateral) spherical triangles [S1, unofficial]. These triangles serve as the starting tessellation of the sphere [S1, unofficial]. The vertices of these triangles are the initial quantized gradient directions [S2, unofficial].
3. **Recursive Subdivision:** For each level of refinement from $1$ to $k$: For each triangle in the current set, calculate the midpoints of the three edges connecting the triangle's vertices, normalize these midpoint vectors to lie on the unit sphere, partition the original triangle into four new triangles by connecting the three normalized midpoints to each other and to the original vertices, and replace the original triangle with these four new triangles [S1, unofficial][S2, unofficial]. After $k$ levels, the set of all triangle vertices constitutes the final quantized gradient directions [S2, unofficial].

**Formula:**
The number of unique vertices $g$ after $k$ levels of refinement is given by:
$$ g = 30 \cdot 4^k + 2 $$
Where:
- $g$ is the total number of quantized gradient directions (vertices) [S1, unofficial].
- $k$ is the number of refinement levels [S1, unofficial].

**Parameters:**
- $k$: Number of refinement levels. Typical value: $k=4$, yielding $g = 30 \cdot 4^4 + 2 = 7682$ vertices [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Non-Congruence on Sphere:** Unlike planar subdivision, the new triangles formed on the sphere are not congruent to each other or to the original triangles [S1, unofficial].
- **Distortion:** While the starting triangles are more nearly planar than those from regular polyhedra (like the octahedron), some distortion still occurs during subdivision [S2, unofficial]. This distortion is significantly less than in octahedron-based methods, which retain 90° angles and cause uneven distribution [S1, unofficial].
- **Vertex Uniqueness:** When calculating midpoints, ensure that vertices shared by adjacent triangles are treated as the same point to maintain the integrity of the mesh structure [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize(v):
    """Normalize a 3D vector to unit length."""
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if length == 0:
        return (0.0, 0.0, 0.0)
    return (v[0]/length, v[1]/length, v[2]/length)

def get_icosahedron_vertices():
    """Return the 12 vertices of a regular icosahedron inscribed in a unit sphere."""
    phi = (1 + math.sqrt(5)) / 2
    vertices = [
        (0, 1, phi), (0, -1, phi), (0, 1, -phi), (0, -1, -phi),
        (1, phi, 0), (-1, phi, 0), (1, -phi, 0), (-1, -phi, 0),
        (phi, 0, 1), (phi, 0, -1), (-phi, 0, 1), (-phi, 0, -1)
    ]
    return [normalize(v) for v in vertices]

def get_dodecahedron_vertices():
    """Return the 20 vertices of a regular dodecahedron inscribed in a unit sphere."""
    phi = (1 + math.sqrt(5)) / 2
    inv_phi = 1 / phi
    vertices = [
        (1, 1, 1), (1, 1, -1), (1, -1, 1), (1, -1, -1),
        (-1, 1, 1), (-1, 1, -1), (-1, -1, 1), (-1, -1, -1),
        (0, inv_phi, phi), (0, -inv_phi, phi), (0, inv_phi, -phi), (0, -inv_phi, -phi),
        (inv_phi, phi, 0), (-inv_phi, phi, 0), (inv_phi, -phi, 0), (-inv_phi, -phi, 0),
        (phi, 0, inv_phi), (-phi, 0, inv_phi), (phi, 0, -inv_phi), (-phi, 0, -inv_phi)
    ]
    return [normalize(v) for v in vertices]

def get_midpoint(v1, v2):
    """Calculate the midpoint of two 3D vectors and normalize it."""
    return normalize((v1[0] + v2[0], v1[1] + v2[1], v1[2] + v2[2]))

def get_initial_triangles(verts):
    # The sources do not provide the exact index list for the 60 triangles.
    # In practice, this is constructed from the known faces of the icosahedron and dodecahedron.
    # Returns a list of 60 triangles, each as a tuple of 3 vertex indices.
    return []

def gradient_quantization(k):
    """
    Perform gradient quantization via icosahedron-dodecahedron tesselation.
    
    Args:
        k (int): Number of refinement levels.
        
    Returns:
        list of tuples: A list of unique unit vectors representing quantized gradients.
    """
    ico_verts = get_icosahedron_vertices()
    dod_verts = get_dodecahedron_vertices()
    
    # Combine and deduplicate vertices
    all_verts = ico_verts + dod_verts
    unique_verts = []
    for v in all_verts:
        if not any(math.isclose(v[i], u[i], rel_tol=1e-9) for i in range(3)):
            unique_verts.append(v)
            
    # Get initial 60 triangles
    triangles = get_initial_triangles(unique_verts)
    
    # Vertex index map for merging shared midpoints
    vert_map = {v: i for i, v in enumerate(unique_verts)}
    
    for level in range(k):
        new_triangles = []
        for tri in triangles:
            v0, v1, v2 = unique_verts[tri[0]], unique_verts[tri[1]], unique_verts[tri[2]]
            
            m01 = get_midpoint(v0, v1)
            m12 = get_midpoint(v1, v2)
            m02 = get_midpoint(v0, v2)
            
            # Map or add midpoints
            if m01 not in vert_map:
                vert_map[m01] = len(unique_verts)
                unique_verts.append(m01)
            if m12 not in vert_map:
                vert_map[m12] = len(unique_verts)
                unique_verts.append(m12)
            if m02 not in vert_map:
                vert_map[m02] = len(unique_verts)
                unique_verts.append(m02)
                
            i0, i1, i2 = tri[0], tri[1], tri[2]
            im01, im12, im02 = vert_map[m01], vert_map[m12], vert_map[m02]
            
            new_triangles.append((i0, im01, im02))
            new_triangles.append((im01, i1, im12))
            new_triangles.append((im02, im12, i2))
            new_triangles.append((im01, im12, im02))
            
        triangles = new_triangles
        
    return unique_verts
```

## Subtopics

### [Octahedral Direction Encoding for Gradient Storage](10730-octahedral-direction-encoding-for-gradient-storage.md)
Octahedral Direction Encoding is a technique to reduce memory usage for precomputed volume gradients by quantizing direction using uniform fractal subdivision of an octahedron into a sphere, rather than storing raw floating-point vectors. It solves the problem of excessive memory consumption in gradient-based volume rendering methods, where storing full 3D gradient vectors for every voxel is prohibitive. It sets itself apart from naive component-wise quantization by providing a uniform distribution of directions on the sphere, avoiding the clustering artifacts inherent in dividing bits among three independent components.

### [Gradient Quantization for Shading](14556-gradient-quantization-for-shading.md)
Gradient Quantization for Shading is a method that quantizes gradient direction vectors to a limited bit depth (typically 12–16 bits) to enable shading computation via a single table lookup, trading accuracy for rendering performance. It distinguishes itself from its parent method, Gradient Quantization via Polyhedron Tesselation, by focusing specifically on the optimization of the shading term in non-photorealistic volume rendering, and from its sibling Octahedral Direction Encoding for Gradient Storage by utilizing a polyhedral tessellation approach rather than octahedral projection.

## Operating characteristics
- **Time Complexity:** The number of vertices grows as $O(4^k)$, and the number of triangles grows as $O(4^k)$. Subdivision at each level requires processing all triangles from the previous level [G].
- **Memory Complexity:** The storage required for the quantized gradients is proportional to the number of vertices $g = 30 \cdot 4^k + 2$. Each vertex requires 3 floating-point numbers [G].
- **Measured Performance:** For $k=4$, the method yields 7682 vertices, which is sufficient for high-quality directional lighting in volume rendering [S1, unofficial].
- **Strengths:** Provides a more uniform distribution of gradient directions compared to octahedron-based methods, reducing artifacts in directional lighting [S1, unofficial][S2, unofficial].
- **Limitations:** The method is static and does not adapt to the specific content of the volume data. The number of gradients is fixed for a given $k$ [G].
- **When to Use:** When high-quality directional lighting is required in volume rendering and a uniform distribution of gradient directions is important [G].
- **When Not to Use:** When memory is extremely constrained and a coarser quantization is acceptable, or when the volume data has specific directional features that could be better captured by a data-dependent quantization method [G].

## Background
The method was introduced in the paper "Direct Volume Rendering with Shading via Three-Dimensional Textures" by Kindlmann et al. (1997) [S1, unofficial][S2, unofficial]. It was developed to address the limitations of previous volume rendering methods that did not support directional lighting effects [S1, unofficial][S2, unofficial]. The authors noted that octahedron-based subdivision resulted in uneven distribution of gradients due to the retention of 90° angles, and the icosahedron-based subdivision had some discrepancy in angles (72° to 60°) [S1, unofficial]. The combination of the icosahedron and dodecahedron was chosen to provide a more uniform starting tessellation with triangles that are more nearly planar, thus undergoing less distortion during subdivision [S1, unofficial][S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970021689: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970021689, fetched 2026-10-26 (unofficial)
[S2] o,,ect Volume Rendering with Shading via Three- ..., https://ntrs.nasa.gov/api/citations/19970021689/downloads/19970021689.pdf?attachment=true, fetched 2026-10-29 (unofficial)
