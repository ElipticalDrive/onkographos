# Gradient Quantization via Polyhedron Tesselation

Also called: Gradient Quantization via Icosahedron-Dodecahedron Tesselation, Icosahedron-Dodecahedron Gradient Quantization

## Overview
Gradient Quantization via Polyhedron Tesselation quantizes the space of gradient directions into a finite set of unit vectors by tessellating a unit sphere with triangles derived from an icosahedron and its dual, the dodecahedron [S1, unofficial][S2, unofficial]. It solves the problem of uneven directional distribution in precomputed gradient storage, which is critical for directional lighting effects in volume rendering [S1, unofficial][S2, unofficial]. It sets itself apart from octahedron-based subdivision by using starting triangles that are more nearly planar, thereby undergoing less distortion during recursive subdivision and avoiding the clustering artifacts caused by the octahedron's persistent 90° angles [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: The number of refinement levels $k$ (integer $\ge 0$).
*   Output: A set of $g$ unit vectors representing quantized gradient directions, stored as vertices on the unit sphere.

**Data Structures:**
The method generates a list of 3D unit vectors. The number of vectors $g$ is determined by the formula below. No additional metadata (such as adjacency lists) is required for the quantization step itself, though the resulting vertices are implicitly connected by the tessellation structure.

**Steps:**
1.  **Initialize Base Vertices:** Construct the vertices of a regular icosahedron (12 vertices) and a regular dodecahedron (20 vertices). These two polyhedra are duals. The combined set contains 32 unique vertices [S1, unofficial][S2, unofficial].
2.  **Form Initial Triangles:** Connect these 32 vertices to form 60 congruent, but not quite equilateral, spherical triangles [S1, unofficial][S2, unofficial]. These triangles serve as the starting tessellation of the unit sphere.
3.  **Recursive Subdivision:** For each level of refinement from $1$ to $k$:
    *   For every triangle in the current set of triangles:
        *   Calculate the midpoints of the three edges.
        *   Project these midpoints onto the unit sphere (normalize the 3D coordinates to length 1).
        *   Partition the original triangle into four new triangles by connecting the original vertices to the new midpoint vertices.
    *   Replace the current set of triangles with the newly generated set.
    *   *Note:* On the sphere, the new triangles are not congruent to each other or to the previous generation, unlike planar subdivision [S1, unofficial][S2, unofficial].
4.  **Extract Quantized Gradients:** The vertices of the final set of triangles constitute the quantized gradient directions. These are unit vectors pointing from the origin to the sphere's surface.

**Formula:**
The total number of quantized gradient vertices $g$ after $k$ levels of refinement is given by:
$$ g = 30 \cdot 4^k + 2 $$
Where:
*   $g$ is the number of vertices (quantized gradients).
*   $k$ is the number of refinement levels.

**Parameters:**
*   $k$: Number of refinement levels.
    *   Typical value: $k=4$, yielding $g = 30 \cdot 4^4 + 2 = 7682$ vertices [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Projecting midpoints onto the sphere requires normalization. Floating-point errors in normalization can accumulate over multiple levels of recursion. It is advisable to re-normalize vertices periodically or use high-precision arithmetic for deep refinements [G].
*   **Degenerate Input:** $k=0$ yields the base 60 triangles and 32 vertices. The formula holds for $k \ge 0$.
*   **Boundary:** The method assumes a unit sphere centered at the origin.

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

def midpoint(v1, v2):
    """Calculate the midpoint of two 3D vectors and project to unit sphere."""
    mx = (v1[0] + v2[0]) / 2.0
    my = (v1[1] + v2[1]) / 2.0
    mz = (v1[2] + v2[2]) / 2.0
    return normalize((mx, my, mz))

def get_icosahedron_vertices():
    """Return the 12 vertices of a regular icosahedron."""
    # Standard coordinates for icosahedron
    t = (1 + math.sqrt(5)) / 2
    vertices = [
        (0, 1, t), (0, -1, t), (0, 1, -t), (0, -1, -t),
        (1, t, 0), (-1, t, 0), (1, -t, 0), (-1, -t, 0),
        (t, 0, 1), (t, 0, -1), (-t, 0, 1), (-t, 0, -1)
    ]
    return [normalize(v) for v in vertices]

def get_dodecahedron_vertices():
    """Return the 20 vertices of a regular dodecahedron."""
    # Standard coordinates for dodecahedron
    t = (1 + math.sqrt(5)) / 2
    vertices = [
        (1, 1, 1), (1, 1, -1), (1, -1, 1), (1, -1, -1),
        (-1, 1, 1), (-1, 1, -1), (-1, -1, 1), (-1, -1, -1),
        (0, t, 1/t), (0, t, -1/t), (0, -t, 1/t), (0, -t, -1/t),
        (1/t, 0, t), (-1/t, 0, t), (1/t, 0, -t), (-1/t, 0, -t),
        (t, 1/t, 0), (-t, 1/t, 0), (t, -1/t, 0), (-t, -1/t, 0)
    ]
    return [normalize(v) for v in vertices]

def generate_gradient_quantization(k):
    """
    Generate quantized gradient directions using icosahedron-dodecahedron tesselation.
    
    Args:
        k: Number of refinement levels.
        
    Returns:
        list of tuples: A list of unique unit vectors representing quantized gradients.
    """
    # Step 1: Initialize Base Vertices
    ico_verts = get_icosahedron_vertices()
    dod_verts = get_dodecahedron_vertices()
    
    # Combine and deduplicate vertices
    all_verts = ico_verts + dod_verts
    unique_verts = []
    seen = set()
    for v in all_verts:
        # Use a tuple representation for hashing
        key = (round(v[0], 10), round(v[1], 10), round(v[2], 10))
        if key not in seen:
            seen.add(key)
            unique_verts.append(v)
            
    # Step 2: Form Initial Triangles
    # The 32 vertices form 60 triangles. 
    # The specific connectivity is defined by the geometry of the dual polyhedra.
    # For implementation, we need the initial list of triangles (triples of vertex indices).
    # Note: The exact index mapping depends on the specific ordering of vertices.
    # Here we assume a helper function that returns the 60 initial triangle indices.
    initial_triangles = get_initial_60_triangles(unique_verts)
    
    current_triangles = initial_triangles
    
    # Step 3: Recursive Subdivision
    for level in range(k):
        new_triangles = []
        # Map old vertices to new midpoints to avoid duplicates
        midpoint_cache = {}
        
        for tri in current_triangles:
            v0, v1, v2 = tri
            
            # Calculate midpoints
            m01_key = (min(v0, v1), max(v0, v1))
            m12_key = (min(v1, v2), max(v1, v2))
            m20_key = (min(v2, v0), max(v2, v0))
            
            if m01_key not in midpoint_cache:
                midpoint_cache[m01_key] = midpoint(unique_verts[v0], unique_verts[v1])
                unique_verts.append(midpoint_cache[m01_key])
                m01_idx = len(unique_verts) - 1
            else:
                # Find index of existing midpoint
                m01_idx = unique_verts.index(midpoint_cache[m01_key])
                
            if m12_key not in midpoint_cache:
                midpoint_cache[m12_key] = midpoint(unique_verts[v1], unique_verts[v2])
                unique_verts.append(midpoint_cache[m12_key])
                m12_idx = len(unique_verts) - 1
            else:
                m12_idx = unique_verts.index(midpoint_cache[m12_key])
                
            if m20_key not in midpoint_cache:
                midpoint_cache[m20_key] = midpoint(unique_verts[v2], unique_verts[v0])
                unique_verts.append(midpoint_cache[m20_key])
                m20_idx = len(unique_verts) - 1
            else:
                m20_idx = unique_verts.index(midpoint_cache[m20_key])
                
            # Partition into 4 new triangles
            # Original: (v0, v1, v2)
            # New: (v0, m01, m20), (v1, m12, m01), (v2, m20, m12), (m01, m12, m20)
            new_triangles.append((v0, m01_idx, m20_idx))
            new_triangles.append((v1, m12_idx, m01_idx))
            new_triangles.append((v2, m20_idx, m12_idx))
            new_triangles.append((m01_idx, m12_idx, m20_idx))
            
        current_triangles = new_triangles
        
    # Step 4: Extract Quantized Gradients
    # The vertices of the final triangles are the quantized gradients.
    # Since we added vertices to unique_verts, we return all unique vertices.
    return unique_verts

def get_initial_60_triangles(vertices):
    """
    Returns the indices of the 60 triangles formed by the 32 vertices of 
    the icosahedron and dodecahedron.
    This is a static lookup table based on the standard vertex ordering.
    """
    # This function would contain the hardcoded indices of the 60 triangles.
    # The exact indices depend on the order of vertices returned by 
    # get_icosahedron_vertices and get_dodecahedron_vertices.
    # For brevity, we return an empty list as a placeholder for the specific topology.
    # In a real implementation, this list must be populated with the correct indices.
    return []
```

## Subtopics

### [Octahedral Direction Encoding for Gradient Storage](10730-octahedral-direction-encoding-for-gradient-storage.md)
Octahedral Direction Encoding is a technique to reduce memory usage for precomputed volume gradients by quantizing direction using uniform fractal subdivision of an octahedron into a sphere, rather than storing raw floating-point vectors. It solves the problem of excessive memory consumption in precomputed radiance transfer or similar gradient-based volume rendering methods, where storing full 3D gradient vectors for every voxel is prohibitive. It sets itself apart from naive component-wise quantization by providing a uniform distribution of directions on the sphere, avoiding the clustering artifacts inherent in dividing bits among three independent components.

## Operating characteristics
*   **Time Complexity:** The number of triangles grows by a factor of 4 at each level. The number of vertices $g$ grows as $O(4^k)$. Generating the vertices requires $O(g)$ operations.
*   **Memory Complexity:** Storing $g$ unit vectors requires $3 \cdot g$ floating-point numbers. For $k=4$, $g=7682$, requiring approximately 92 KB for single-precision floats.
*   **Measured Performance:** For $k=4$, the method yields 7682 vertices [S1, unofficial][S2, unofficial].
*   **Strengths:** Provides a more uniform distribution of gradient directions compared to octahedron-based methods. The starting triangles are more nearly planar, reducing distortion during subdivision [S1, unofficial][S2, unofficial].
*   **Limitations:** The triangles are not perfectly equilateral, leading to some variation in angular distribution. The method is static; it does not adapt to the specific gradient distribution of the volume data.
*   **When to Use:** When high-quality directional lighting is required in volume rendering and memory constraints allow for a moderate number of gradient directions (e.g., ~7000).
*   **When Not to Use:** When memory is extremely constrained, or when the gradient distribution is highly anisotropic and adaptive quantization is needed.

## Background
The method was developed for direct volume rendering with shading via three-dimensional textures [S1, unofficial][S2, unofficial]. It addresses the limitation of earlier methods that did not allow for directional lighting effects [S2, unofficial]. The authors proposed this method as an improvement over octahedron and icosahedron-only tessellations, leveraging the dual relationship between the icosahedron and dodecahedron to create a more uniform starting set of triangles [S1, unofficial][S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970021689: Direct Volume Rendering with Shading via Three-Dimensional Textures, https://archive.org/details/NASA_NTRS_Archive_19970021689, fetched 2026-10-26 (unofficial)
[S2] o,,ect Volume Rendering with Shading via Three- ..., https://ntrs.nasa.gov/api/citations/19970021689/downloads/19970021689.pdf?attachment=true, fetched 2026-10-29 (unofficial)
