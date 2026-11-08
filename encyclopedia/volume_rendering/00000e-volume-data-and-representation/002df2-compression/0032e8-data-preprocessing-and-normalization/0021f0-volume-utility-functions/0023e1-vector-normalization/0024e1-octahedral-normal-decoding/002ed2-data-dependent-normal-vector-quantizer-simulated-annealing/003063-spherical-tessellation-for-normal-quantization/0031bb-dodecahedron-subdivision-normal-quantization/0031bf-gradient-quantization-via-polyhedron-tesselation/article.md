# Gradient Quantization via Polyhedron Tesselation

Also called: Gradient Quantization via Icosahedron-Dodecahedron Tesselation, Icosahedron-Dodecahedron Gradient Quantization

## Overview
Gradient Quantization via Polyhedron Tesselation quantizes the 3D gradient direction space into a finite set of unit vectors by tessellating a unit sphere with triangles derived from an icosahedron and its dual, the dodecahedron [S1, S2, unofficial]. It solves the problem of uneven distribution of normal vectors for directional lighting in volume rendering, which arises when using simpler polyhedra like the octahedron [S1, S2, unofficial]. It sets itself apart from octahedron-based methods by using triangles that are more nearly planar and undergo less distortion during subdivision, avoiding the retention of 90° angles that cause uneven gradient distribution [S1, S2, unofficial].

## How it works
**Inputs:**
- Refinement level $k$ (integer $\ge 0$).

**Outputs:**
- A set of $g$ unit vectors representing quantized gradient directions, where $g = 30 \cdot 4^k + 2$ [S1, S2, unofficial].

**Data Structures:**
- The algorithm generates vertices on a unit sphere. No specific bit-packing is defined in the sources; the output is conceptually a list of 3D coordinates $(x, y, z)$ normalized to unit length.

**Steps:**
1. **Initialization:** Start with the combined vertices of a regular icosahedron (12 vertices) and a regular dodecahedron (20 vertices) [S1, S2, unofficial]. These 32 vertices define 60 congruent, but not quite equilateral, triangles on the sphere [S1, S2, unofficial].
2. **Recursive Subdivision:** For each level of refinement from 1 to $k$:
   - For every triangle in the current set:
     - Calculate the midpoints of the three sides of the triangle.
     - Project these midpoints onto the unit sphere (normalize the midpoint vectors) [S1, S2, unofficial].
     - Partition the original triangle into four new triangles using the three original vertices and the three new midpoint vertices [S1, S2, unofficial].
3. **Result:** After $k$ levels, the vertices of the resulting triangles constitute the quantized gradient directions [S1, S2, unofficial].

**Formula:**
The number of vertices $g$ after $k$ levels of refinement is given by:
$$ g = 30 \cdot 4^k + 2 $$
For $k=4$, this yields 7682 vertices [S1, S2, unofficial].

**Parameters:**
- $k$: Number of refinement levels. A typical value is 4, yielding 7682 vertices [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- On the sphere, unlike the plane, the new triangles produced by subdivision are not congruent [S1, S2, unofficial].
- The starting triangles are "not quite equilateral" [S1, S2, unofficial].
- The method avoids the distortion issues of octahedron subdivision, where triangles retain 90° angles while others approach 60°, leading to uneven distribution [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize(v):
    """Normalize a 3D vector to unit length."""
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if length == 0:
        return (0, 0, 0)
    return (v[0]/length, v[1]/length, v[2]/length)

def midpoint(v1, v2):
    """Calculate the midpoint of two 3D vectors."""
    return ((v1[0] + v2[0])/2, (v1[1] + v2[1])/2, (v1[2] + v2[2])/2)

def get_icosahedron_vertices():
    """
    Returns the 12 vertices of a regular icosahedron centered at origin.
    Exact coordinates not provided in sources; assumed to be standard unit icosahedron vertices.
    """
    # Placeholder for standard icosahedron vertex generation
    # The sources do not provide the exact coordinates, only the count (12).
    # An implementer would need to generate standard icosahedron vertices.
    # This is a gap filled by general knowledge [G].
    phi = (1 + math.sqrt(5)) / 2
    vertices = [
        (0, 1, phi), (0, -1, phi), (0, 1, -phi), (0, -1, -phi),
        (1, phi, 0), (-1, phi, 0), (1, -phi, 0), (-1, -phi, 0),
        (phi, 0, 1), (-phi, 0, 1), (phi, 0, -1), (-phi, 0, -1)
    ]
    return [normalize(v) for v in vertices]

def get_dodecahedron_vertices():
    """
    Returns the 20 vertices of a regular dodecahedron centered at origin.
    Exact coordinates not provided in sources; assumed to be standard unit dodecahedron vertices.
    """
    # Placeholder for standard dodecahedron vertex generation
    # The sources do not provide the exact coordinates, only the count (20).
    # An implementer would need to generate standard dodecahedron vertices.
    # This is a gap filled by general knowledge [G].
    phi = (1 + math.sqrt(5)) / 2
    vertices = [
        (1, 1, 1), (1, 1, -1), (1, -1, 1), (1, -1, -1),
        (-1, 1, 1), (-1, 1, -1), (-1, -1, 1), (-1, -1, -1),
        (0, phi, 1/phi), (0, phi, -1/phi), (0, -phi, 1/phi), (0, -phi, -1/phi),
        (1/phi, 0, phi), (-1/phi, 0, phi), (1/phi, 0, -phi), (-1/phi, 0, -phi),
        (phi, 1/phi, 0), (-phi, 1/phi, 0), (phi, -1/phi, 0), (-phi, -1/phi, 0)
    ]
    return [normalize(v) for v in vertices]

def get_initial_triangles():
    """
    Returns the 60 triangles formed by the combined icosahedron and dodecahedron vertices.
    The sources state these 32 vertices produce 60 congruent triangles.
    The exact connectivity is not provided in the sources.
    This is a gap filled by general knowledge [G].
    """
    # The sources do not provide the face connectivity for the combined polyhedron.
    # An implementer would need to compute the convex hull or use a known mesh
    # for the icosidodecahedron (which has 32 vertices and 60 faces).
    # This is a gap filled by general knowledge [G].
    pass

def gradient_quantization(k):
    """
    Performs gradient quantization via polyhedron tessellation.
    
    Args:
        k: Number of refinement levels.
        
    Returns:
        A list of unique unit vectors representing quantized gradients.
    """
    # Step 1: Get initial vertices
    ico_verts = get_icosahedron_vertices()
    dod_verts = get_dodecahedron_vertices()
    all_verts = ico_verts + dod_verts
    
    # Step 2: Get initial triangles
    # Note: The sources do not provide the specific triangle indices.
    # We assume a function exists that returns the 60 triangles from the 32 vertices.
    triangles = get_initial_triangles()
    
    # Step 3: Recursive subdivision
    for level in range(k):
        new_triangles = []
        for t in triangles:
            v0, v1, v2 = t
            # Calculate midpoints
            m01 = normalize(midpoint(v0, v1))
            m12 = normalize(midpoint(v1, v2))
            m20 = normalize(midpoint(v2, v0))
            
            # Create 4 new triangles
            # Triangle 1: v0, m01, m20
            new_triangles.append((v0, m01, m20))
            # Triangle 2: m01, v1, m12
            new_triangles.append((m01, v1, m12))
            # Triangle 3: m20, m12, v2
            new_triangles.append((m20, m12, v2))
            # Triangle 4: m01, m12, m20
            new_triangles.append((m01, m12, m20))
        
        triangles = new_triangles
    
    # Step 4: Collect all unique vertices
    unique_verts = set()
    for t in triangles:
        unique_verts.add(t[0])
        unique_verts.add(t[1])
        unique_verts.add(t[2])
        
    return list(unique_verts)
```

## Operating characteristics
- **Time Complexity:** Not explicitly given, but depends on $k$. The number of triangles grows by a factor of 4 each level.
- **Memory Complexity:** Not explicitly given, but proportional to the number of vertices $g = 30 \cdot 4^k + 2$ [S1, S2, unofficial].
- **Measured Performance:** For $k=4$, 7682 vertices are generated [S1, S2, unofficial].
- **Strengths:** Produces more evenly distributed gradients than octahedron-based methods because the starting triangles are more nearly planar and undergo less distortion during subdivision [S1, S2, unofficial].
- **Limitations:** The starting triangles are not quite equilateral, and on the sphere, the subdivided triangles are not congruent [S1, S2, unofficial].
- **When to use:** When directional lighting effects are needed in volume rendering and a more uniform distribution of gradient directions is required compared to octahedron-based methods [S1, S2, unofficial].
- **When not to use:** When computational resources are extremely limited and a coarser quantization is acceptable, or when the specific distortion of octahedron subdivision is not a concern.

## Background
- **Origin:** Developed by researchers in 1997 for direct volume rendering with shading via three-dimensional textures [S1, S2, unofficial].
- **History:** The method was introduced to address the limitations of previous methods that did not allow for directional lighting effects [S1, S2, unofficial]. It improves upon octahedron and icosahedron-based tessellations by combining the icosahedron and its dual, the dodecahedron [S1, S2, unofficial].
- **Usage:** Used in volume rendering to quantize gradient directions for lighting calculations [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970021689: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970021689, fetched 2026-10-26 (unofficial)
[S2] Object Volume Rendering with Shading via Three- ..., https://ntrs.nasa.gov/api/citations/19970021689/downloads/19970021689.pdf?attachment=true, fetched 2026-10-29 (unofficial)
