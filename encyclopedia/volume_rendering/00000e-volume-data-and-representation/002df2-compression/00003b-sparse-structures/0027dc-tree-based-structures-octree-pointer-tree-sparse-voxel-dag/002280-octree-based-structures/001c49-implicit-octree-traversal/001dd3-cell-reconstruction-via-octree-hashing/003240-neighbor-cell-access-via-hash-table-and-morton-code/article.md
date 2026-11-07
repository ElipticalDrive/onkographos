# Neighbor Cell Access via Hash Table and Morton Code

Also called: Adjacent Cell Retrieval

## Overview
This method retrieves face-adjacent cells in an irregular (generalized) octree where subdivision is not uniform across all three axial directions. It solves the problem that standard Morton code arithmetic fails to identify true geometric neighbors in anisotropic structures, where cells close in Morton space may be far apart geometrically. It sets itself apart from sibling methods by explicitly validating adjacency through geometric limits rather than relying solely on bitwise offsets or parent pointers, ensuring correctness despite irregular subdivision [S1].

## How it works
The method uses a hash table to store octree cells indexed by their Morton codes, enabling near-constant time lookups. Each cell stores its spatial coordinates to complement the Morton code, which is insufficient for irregular grids alone [S1].

**Inputs:**
*   Current cell $c$.
*   Target face $f$ (direction to search).

**Outputs:**
*   Face-adjacent cell $a$ through face $f$, or null if none exists.

**Data Structures:**
*   **Hash Table:** Maps Morton codes to cell objects.
*   **Cell Object:** Contains:
    *   `morton_code`: Integer representing spatial position.
    *   `level`: Depth in the octree.
    *   `spatial_coords`: Geometric bounds or center (used to determine limits).

**Algorithm Steps:**

1.  **Initialize:**
    *   Get the level of cell $c$: $L_c = \text{GetLevel}(c)$.
    *   Determine the coincident face $f'$ on the neighbor side (opposite to $f$).
    *   Set search offset $k = 1$.

2.  **Iterative Search:**
    *   While $k \le L_c$:
        a.  **Candidate Generation:** Call $\text{GetAdjacentCellByFace}(c, f, k)$.
            *   Retrieve spatial coordinates of $c$.
            *   Add offset $k$ in the direction of face $f$.
            *   Convert the resulting spatial coordinate to a Morton code $M_a$.
            *   Look up $M_a$ in the hash table. Let the result be cell $a$.
        b.  **Existence Check:**
            *   If $a$ does not exist in the hash table, increment $k$ ($k \leftarrow k + 1$) and continue.
        c.  **Geometric Validation:**
            *   If $a$ exists:
                i.  Get the geometric limit of $c$ in direction $f$: $Limit_c = \text{GetCellLimitByFace}(c, f)$.
                ii. Get the geometric limit of $a$ in direction $f'$: $Limit_a = \text{GetCellLimitByFace}(a, f')$.
                iii. If $Limit_c == Limit_a$, the cells share a true geometric boundary. Return $a$.
                iv. Else, the cells are adjacent in Morton space but not geometrically (due to irregular subdivision). Increment $k$ ($k \leftarrow k + 1$) and continue.

3.  **Fallback:**
    *   If the loop completes without finding a valid neighbor, the neighbor is likely the parent cell.
    *   Shift the three least significant bits of $c$'s Morton code to obtain the parent's Morton code.
    *   Return the parent cell [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_neighbor(cell_c, face_f):
    """
    Finds the face-adjacent neighbor of cell_c across face_f.
    Handles anisotropic octrees by validating geometric limits.
    """
    level_c = get_level(cell_c)
    face_f_prime = get_coincident_face(face_f)
    offset = 1
    
    while offset <= level_c:
        # 1. Generate candidate via spatial offset and Morton conversion
        candidate_morton = get_adjacent_cell_by_face_morton(cell_c, face_f, offset)
        
        # 2. Hash table lookup
        candidate_cell = hash_table.get(candidate_morton)
        
        if candidate_cell is not None:
            # 3. Geometric validation
            limit_c = get_cell_limit_by_face(cell_c, face_f)
            limit_a = get_cell_limit_by_face(candidate_cell, face_f_prime)
            
            if limit_c == limit_a:
                return candidate_cell
        
        offset += 1
        
    # 4. Fallback to parent if no direct neighbor found at current level
    # Shift 3 bits to get parent Morton code
    parent_morton = cell_c.morton_code >> 3
    return hash_table.get(parent_morton)

def get_adjacent_cell_by_face_morton(cell, face, offset):
    """
    Calculates the Morton code of a cell offset by 'offset' units 
    in the direction of 'face' from 'cell'.
    """
    # Get spatial coordinates (e.g., min corner) of cell
    coords = cell.spatial_coords
    
    # Add offset in the specific axial direction defined by face
    # face is typically an enum or index (0=x, 1=y, 2=z)
    coords[face] += offset
    
    # Convert spatial coordinates to Morton code
    # Assumes a regular grid mapping for the coordinate system
    return spatial_to_morton(coords)

def get_cell_limit_by_face(cell, face):
    """
    Returns the geometric boundary coordinate of the cell 
    in the direction of the given face.
    """
    # For a face-aligned boundary, this is typically the max coordinate 
    # in the direction of the face (if face points outward)
    # or min coordinate (if face points inward).
    # Assuming face_f points from cell to neighbor:
    return cell.bounds.max[face] # [G] Implementation detail: bounds structure assumed
```

**Parameters:**
*   **Offset Limit:** The search offset is bounded by the current cell's level ($L_c$). Anisotropic subdivision typically occurs at deeper levels, ensuring the offset does not exceed the depth [S1].

**Edge Cases:**
*   **Irregular Subdivision:** Cells may be close in Morton code but far in geometry. The geometric limit check prevents returning false positives [S1].
*   **Boundary Conditions:** If the neighbor is outside the domain, the hash table lookup returns null, and the loop continues or falls back to the parent (if within domain) [S1].

## Subtopics

### Generalized Octree with Morton Codes
A space subdivision data structure where octree cells do not require subdivision in all three main axial directions; they may be subdivided in only one or two directions to adapt to anisotropic volumetric data [S1]. To access neighbors, the algorithm relies on spatial coordinates stored within cells. Sub-cells contained inside a parent cell must tile the parent's space, meaning all Morton codes indexing sub-cells point to valid cells within that space. This allows neighbor access even with irregular division [S1]. The data structure is a hashed octree using Morton codes as indices, augmented with spatial coordinates [S1].

## Operating characteristics
*   **Time Complexity:** Near-constant time $O(1)$ for hash table lookups. The neighbor search loop runs at most $L_c$ times, where $L_c$ is the cell level. In practice, the first offset ($k=1$) often suffices, making it effectively $O(1)$ [S1].
*   **Memory Complexity:** Requires storing spatial coordinates for each cell in addition to the Morton code and hash table overhead.
*   **Strengths:** Correctly handles anisotropic/irregular octrees where standard Morton arithmetic fails. Robust against non-manifold or complex surface configurations by validating geometric limits [S1].
*   **Limitations:** Requires storing extra spatial data per cell. Performance depends on hash table efficiency.
*   **When to Use:** When working with anisotropic volumetric data requiring adaptive, non-uniform octree subdivision [S1].

## Background
This method originates from work on adaptive surface extraction from anisotropic volumetric data, specifically contouring on generalized octrees. It was developed to handle volumetric data where subdivision adapts to local features, leading to irregular tree structures [S1].

## Sources
[S1] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
