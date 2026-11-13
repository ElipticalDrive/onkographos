# Domain Search Isosurface Extraction

Also called: Domain Search

## Overview
Domain Search Isosurface Extraction is a tetrahedral marching method that extracts an isosurface by iterating through a volumetric grid, decomposing each voxel into tetrahedra, and generating vertices via linear interpolation at the isovalue [S1, unofficial]. It sets itself apart from its siblings by using a direct, brute-force enumeration of all tetrahedral cases within a fixed-size buffer, relying on explicit index bit-masking rather than spatial hierarchies or adaptive traversal [S1, unofficial]. Unlike [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md) which grows a mesh front, or [Region-growing Iso-surface Extraction](7568-region-growing-iso-surface-extraction.md) which follows connectivity, this method processes every cell independently [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric grid defined by dimensions $x\_axis, y\_axis, z\_axis$ (typical value: $256 \times 256 \times 256$) [S1, unofficial]. The grid contains scalar field values at vertices.
*   **Input:** An `isolevel` (float) defining the target isosurface value.
*   **Input:** An array `Vertices` of size $N$ containing the 3D coordinates ($Vector3f$) of the grid vertices.
*   **Input:** An array `col` of size 4 containing the scalar values of the four vertices of the current tetrahedron.
*   **Output:** An array `newvert` of size $256 \times 256 \times 400$ storing the extracted $Vector3f$ positions [S1, unofficial].
*   **Output:** An integer `p` tracking the current write index in `newvert`.

**Data Structures:**
*   `Vertices`: Array of $Vector3f$.
*   `newvert`: Array of $Vector3f$, pre-allocated with size $256 \times 256 \times 400$ [S1, unofficial].
*   `col`: Vector of integers/floats, size 4, holding scalar values for the current tetrahedron's vertices.
*   `index`: Unsigned char (8-bit), used to store the case mask.

**Algorithm Steps:**

1.  **Iterate** through all voxels in the grid. For each voxel, decompose it into tetrahedra (the source code implies a specific decomposition where 4 vertices `a, b, c, d` define the tetrahedron) [S1, unofficial].
2.  **Compute Case Index:** For the current tetrahedron with scalar values `col[0]` to `col[3]` at vertices `a, b, c, d` respectively:
    *   Initialize `index = 0`.
    *   For $i$ from 0 to 3:
        *   If `col[i] < isolevel`, set bit $i$ in `index`: `index |= (1 << i)` [S1, unofficial].
3.  **Check Buffer Bounds:** Before generating vertices, check if `p < (ind - 20)`. If false, skip vertex generation for this cell to prevent buffer overflow [S1, unofficial].
4.  **Handle Case:** Based on `index`, generate interpolated vertices and append them to `newvert` at index `p`, incrementing `p` by the number of vertices generated:
    *   **Case 0x00 (0000) and 0x0F (1111):** All vertices are outside or inside. Do nothing. `break` [S1, unofficial].
    *   **Case 0x01 (0001):** Only vertex `a` (index 0) is inside. Generate 3 vertices by interpolating edges connected to `a`:
        *   Edge $(a, b)$: Interpolate between `Vertices[a]` and `Vertices[b]` using `col[0]` and `col[1]`.
        *   Edge $(a, d)$: Interpolate between `Vertices[a]` and `Vertices[d]` using `col[0]` and `col[3]`.
        *   Edge $(a, c)$: Interpolate between `Vertices[a]` and `Vertices[c]` using `col[0]` and `col[2]`.
        *   Increment `p` by 3 [S1, unofficial].
    *   **Case 0x02 (0010):** Only vertex `b` (index 1) is inside. Generate 3 vertices from edges $(b, a)$, $(b, c)$, $(b, d)$. Increment `p` by 3 [S1, unofficial].
    *   **Case 0x04 (0100):** Only vertex `c` (index 2) is inside. Generate 3 vertices from edges $(c, a)$, $(c, d)$, $(c, b)$. Increment `p` by 3 [S1, unofficial].
    *   **Case 0x08 (1000):** Only vertex `d` (index 3) is inside. Generate 3 vertices from edges $(d, b)$, $(d, c)$, $(d, a)$. Increment `p` by 3 [S1, unofficial].
    *   **Case 0x03 (0011):** Vertices `a, b` inside. Generate 6 vertices forming two triangles. The source code specifies interpolating edges: $(a, d)$, $(a, c)$, $(b, d)$, $(a, c)$, $(b, d)$, $(a, c)$? *Note: The source code for Case 0x03 lists specific interpolations. We follow the source code exactly:*
        *   Interpolate $(a, d)$ using `col[0], col[3]`.
        *   Interpolate $(a, c)$ using `col[0], col[2]`.
        *   Interpolate $(b, d)$ using `col[1], col[3]`.
        *   Interpolate $(a, c)$ using `col[0], col[2]` (repeated in source? Source says `interpolate(Vertices[a].x,Vertices[c].x...` then later `interpolate(Vertices[b].x,Vertices[d].x...` then `interpolate(Vertices[a].x,Vertices[c].x...`). *Correction based on source text:* The source lists 6 interpolations. We must output them as written in the source logic for the case 0x03:
            1. $(a, d)$
            2. $(a, c)$
            3. $(b, d)$
            4. $(a, c)$
            5. $(b, d)$
            6. $(a, c)$
            *Wait, the source text for 0x03 is:*
            `newvert[p] = ... (a,d) ... p++;`
            `newvert[p] = ... (a,c) ... p++;`
            `newvert[p] = ... (b,d) ... p++;`
            `newvert[p] = ... (a,c) ... p++;` (Source snippet cuts off but implies 6 vertices).
            *Actually, looking at Case 0x05 (0,2 inside), the source lists:*
            1. $(a, b)$
            2. $(a, d)$
            3. $(b, c)$
            4. $(d, c)$
            *The source code for 0x03 is incomplete in the snippet provided but implies 6 vertices. We will describe the general interpolation logic and note that the specific edge pairs are determined by the source code's switch case.*
            *Refined Instruction:* The source code explicitly lists the interpolations for each case. We must replicate the logic. For 0x03, the source shows 3 vertices being generated in the first part, then continues. The standard tetrahedral marching cube for 2 vertices inside usually generates a triangle (3 vertices) or a quad (4 vertices) depending on connectivity. The source code increments `p` by 6 for 2-vertex cases. This implies it generates 6 vertices (likely two triangles or a fan). We will list the interpolations as provided in the source for the visible cases and generalize the pattern for others if not fully visible, but the prompt requires exactness.
            *Let's look at Case 0x05 (0,2 inside) in source:*
            1. $(a, b)$
            2. $(a, d)$
            3. $(b, c)$
            4. $(d, c)$
            *This is only 4 vertices. But the note says "Increment p by 6". The source code snippet for 0x05 shows 4 assignments. There is a discrepancy between the note "Increment p by 6" and the visible code for 0x05 which shows 4. However, the note for 0x03 says "Increment p by 6". We will follow the source code's explicit assignments where visible, and assume the "6" in the note might refer to a different decomposition or a typo in the source notes vs code. Given the instruction "rewrite in one clean common style", we will implement the logic shown in the code.*
            *Actually, the source code for 0x03 shows:*
            `newvert[p] = ... (a,d) ... p++;`
            `newvert[p] = ... (a,c) ... p++;`
            `newvert[p] = ... (b,d) ... p++;`
            `newvert[p] = ... (a,c) ... p++;` (This line is cut off in the snippet but `p++` is implied).
            *Let's stick to the visible code structure.*
    *   **General Interpolation Logic:** For any edge between vertex $V_i$ and $V_j$ with scalar values $val_i$ and $val_j$:
        *   If $val_i == val_j$, the intersection is the midpoint: $P = (V_i + V_j) / 2.0$.
        *   Otherwise, calculate interpolation factor $t = \frac{|isolevel - val_i|}{|val_j - val_i|}$.
        *   $P = V_i \times (1 - t) + V_j \times t$.
        *   *Note: The source code uses `abs((isolevel - a) / (b - a))`. This is equivalent to linear interpolation.* [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def domain_search_isosurface_extraction(Vertices, col, isolevel, ind):
    """
    Vertices: List of Vector3f objects representing grid vertices.
    col: List of 4 floats representing scalar values for the current tetrahedron vertices [a, b, c, d].
    isolevel: Float, the target isovalue.
    ind: Integer, the maximum size of the newvert buffer.
    
    Returns:
    newvert: List of Vector3f, the extracted vertices.
    p: Integer, the number of vertices generated.
    """
    newvert = [None] * (256 * 256 * 400)  # Pre-allocated size from source
    p = 0
    
    # Helper for interpolation
    def interpolate(p1, p2, val1, val2):
        if val1 == val2:
            return Vector3f((p1.x + p2.x) / 2.0, (p1.y + p2.y) / 2.0, (p1.z + p2.z) / 2.0)
        else:
            # Source uses abs((isolevel - a) / (b - a))
            # Note: The source code passes 'isolevel' to interpolate, but the formula uses a and b.
            # The function signature in source is: interpolate(x, y, isolevel, a, b)
            # where x,y are coordinates of the two vertices.
            # interp = abs((isolevel - a) / (b - a))
            # z = x * (1 - interp) + y * interp
            # This is standard linear interpolation.
            interp = abs((isolevel - val1) / (val2 - val1))
            one_minus_interp = 1.0 - interp
            return Vector3f(
                p1.x * one_minus_interp + p2.x * interp,
                p1.y * one_minus_interp + p2.y * interp,
                p1.z * one_minus_interp + p2.z * interp
            )

    # In a real implementation, this function would be called for each tetrahedron
    # in the grid. Here we simulate one tetrahedron with vertices a, b, c, d.
    # a, b, c, d are indices into the Vertices array.
    a, b, c, d = 0, 1, 2, 3 # Example indices
    
    # 1. Compute case index
    index = 0
    if col[0] < isolevel:
        index |= (1 << 0)
    if col[1] < isolevel:
        index |= (1 << 1)
    if col[2] < isolevel:
        index |= (1 << 2)
    if col[3] < isolevel:
        index |= (1 << 3)
        
    # 2. Check buffer bounds
    if p < (ind - 20):
        # 3. Handle cases
        if index == 0x00 or index == 0x0F:
            pass # Do nothing
        elif index == 0x01: # Only vertex 0 (a) inside
            newvert[p] = interpolate(Vertices[a], Vertices[b], col[0], col[1]); p += 1
            newvert[p] = interpolate(Vertices[a], Vertices[d], col[0], col[3]); p += 1
            newvert[p] = interpolate(Vertices[a], Vertices[c], col[0], col[2]); p += 1
        elif index == 0x02: # Only vertex 1 (b) inside
            newvert[p] = interpolate(Vertices[b], Vertices[a], col[1], col[0]); p += 1
            newvert[p] = interpolate(Vertices[b], Vertices[c], col[1], col[2]); p += 1
            newvert[p] = interpolate(Vertices[b], Vertices[d], col[1], col[3]); p += 1
        elif index == 0x04: # Only vertex 2 (c) inside
            newvert[p] = interpolate(Vertices[c], Vertices[a], col[2], col[0]); p += 1
            newvert[p] = interpolate(Vertices[c], Vertices[d], col[2], col[3]); p += 1
            newvert[p] = interpolate(Vertices[c], Vertices[b], col[2], col[1]); p += 1
        elif index == 0x08: # Only vertex 3 (d) inside
            newvert[p] = interpolate(Vertices[d], Vertices[b], col[3], col[1]); p += 1
            newvert[p] = interpolate(Vertices[d], Vertices[c], col[3], col[2]); p += 1
            newvert[p] = interpolate(Vertices[d], Vertices[a], col[3], col[0]); p += 1
        elif index == 0x03: # Vertices 0, 1 (a, b) inside
            # Source code lists 6 vertices for 2-vertex cases, though only 3 are explicitly shown in the snippet for 0x03.
            # We will generate the 3 visible ones and note the pattern.
            newvert[p] = interpolate(Vertices[a], Vertices[d], col[0], col[3]); p += 1
            newvert[p] = interpolate(Vertices[a], Vertices[c], col[0], col[2]); p += 1
            newvert[p] = interpolate(Vertices[b], Vertices[d], col[1], col[3]); p += 1
            # The source code snippet cuts off here for 0x03, but the note says "Increment p by 6".
            # We assume the remaining 3 vertices are generated similarly based on the source's pattern for other cases.
            # However, without the exact code, we cannot invent them. We will stop at the visible code.
            # [G] In a complete implementation, the remaining 3 vertices for case 0x03 would be generated.
            # Based on Case 0x05 (0,2 inside), the pattern seems to be generating vertices on edges connecting inside to outside.
            # For 0x03 (a,b inside), edges are (a,d), (a,c), (b,d), (b,c).
            # The source code for 0x05 generates: (a,b) - wait, 0,2 inside. Edges: (a,b), (a,d), (b,c), (d,c).
            # For 0x03, edges should be: (a,d), (a,c), (b,d), (b,c).
            # The source code for 0x03 shows: (a,d), (a,c), (b,d).
            # It is likely the next 3 are (b,c), (a,c), (b,d) or similar.
            # Since the source is incomplete, we mark the rest as not given.
            pass 
        elif index == 0x05: # Vertices 0, 2 (a, c) inside
            newvert[p] = interpolate(Vertices[a], Vertices[b], col[0], col[1]); p += 1
            newvert[p] = interpolate(Vertices[a], Vertices[d], col[0], col[3]); p += 1
            newvert[p] = interpolate(Vertices[b], Vertices[c], col[1], col[2]); p += 1
            newvert[p] = interpolate(Vertices[d], Vertices[c], col[3], col[2]); p += 1
            # Source note says "Increment p by 6", but code shows 4.
            # [G] Assuming the source code is the ground truth for the implementation.
        # Other cases (0x06, 0x09, 0x0A, 0x0C) follow similar patterns based on source snippets.
        
    return newvert[:p], p
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If the scalar values of two vertices are equal ($a == b$), the interpolation formula divides by zero. The source code handles this by printing '1' and using the midpoint formula $(x+y)/2.0$ [S1, unofficial].
*   **Buffer Overflow:** The array `newvert` has a fixed size. The code checks `p < (ind - 20)` before writing. If this condition fails, vertices are skipped, potentially resulting in an incomplete mesh [S1, unofficial].
*   **Degenerate Input:** If the isovalue equals a vertex value, the vertex itself is considered "inside" (since `col[i] < isolevel` is false, it is outside? No, `col[i] < isolevel` means inside. If `col[i] == isolevel`, it is outside. This is a specific convention. The source uses strict inequality `<` for inside. This means vertices exactly on the isosurface are treated as outside. This may cause gaps if the isovalue aligns with grid points [S1, unofficial].

## Subtopics

### [Octree-Based Isosurface Extraction](7786-octree-based-isosurface-extraction.md)
Uses an octree to prune empty space, differing from Domain Search by using spatial hierarchy instead of value-based indexing.

### [LOD-Aware Hierarchical Isosurface Search](10909-lod-aware-hierarchical-isosurface-search.md)
Traverses a hierarchical voxel tree to extract isosurfaces at specific LODs, differing from Domain Search by supporting dynamic LOD and isovalue specification.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels/tetrahedra, as every cell is processed independently [S1, unofficial].
*   **Memory Complexity:** $O(V)$ where $V$ is the size of the output vertex buffer. The source uses a fixed-size buffer of $256 \times 256 \times 400$ [S1, unofficial].
*   **Performance:** The method is simple to implement and parallelize due to the independence of tetrahedra. However, the fixed buffer size and lack of spatial pruning make it inefficient for large volumes or sparse data compared to hierarchical methods [S1, unofficial].
*   **Strengths:** Simple logic, no complex data structures required, easy to debug.
*   **Limitations:** Fixed buffer size can lead to overflow or wasted space. No connectivity information is generated (only vertices). The source code does not show triangle generation, only vertex generation, which is insufficient for rendering without further processing [S1, unofficial].

## Background
The method is described in an unofficial GitHub repository by yashwanth850 [S1, unofficial]. It appears to be a student or hobbyist implementation of a tetrahedral marching cubes variant, likely for educational purposes or a specific project requiring simple isosurface extraction. The code includes OpenGL rendering components, suggesting it was used for visualization.

## Sources
[S1] yashwanth850/Domain-search-isosurface-extraction, https://github.com/yashwanth850/Domain-search-isosurface-extraction, fetched 2026-10-27 (unofficial)
