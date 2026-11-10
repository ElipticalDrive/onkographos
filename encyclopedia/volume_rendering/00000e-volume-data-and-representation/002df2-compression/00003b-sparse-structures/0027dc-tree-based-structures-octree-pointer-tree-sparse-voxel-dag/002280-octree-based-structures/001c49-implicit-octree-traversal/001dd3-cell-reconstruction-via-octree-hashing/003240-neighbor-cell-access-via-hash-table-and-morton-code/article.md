# Neighbor Cell Access via Hash Table and Morton Code

Also called: Adjacent Cell Retrieval

## Overview
A method to find face-adjacent cells in an irregular (generalized) octree using spatial coordinates, Morton codes, and a hash table for constant-time access [S1]. It handles anisotropic subdivision by checking geometric limits to ensure true adjacency, distinguishing it from standard octree neighbor traversal which assumes regular subdivision [S1]. It builds on [Cell Reconstruction via Octree Hashing](1595-cell-reconstruction-via-octree-hashing.md) by providing the spatial adjacency logic required for surface extraction on generalized grids.

## How it works
**Inputs:**
- A query cell $c$ (identified by its Morton code and stored spatial metadata).
- A target face $f$ of cell $c$ for which the neighbor is sought.

**Outputs:**
- The face-adjacent cell $a$ sharing face $f$, or `null` if no such cell exists.

**Data Structures:**
- **Hash Table:** Maps Morton codes to cell objects. Provides almost constant-time lookup [S1].
- **Cell Object:** Stores the Morton code and spatial coordinates (e.g., bounding box or center) necessary to resolve geometric adjacency [S1].

**Algorithm Steps:**
1.  **Get Level:** Determine the depth/level of the current cell $c$. Let this be $L_c$.
2.  **Identify Coincident Face:** Determine the geometric face $f_{adj}$ on the potential neighbor that corresponds to the query face $f$ on $c$.
3.  **Initialize Offset:** Set search offset $k = 1$.
4.  **Iterative Search:** While $k \le L_c$:
    a.  **Compute Candidate Morton Code:**
        i.  Retrieve the spatial coordinates of cell $c$.
        ii. Add an offset vector of magnitude $k$ in the direction normal to face $f$.
        iii. Convert these new spatial coordinates into a Morton code $M_{cand}$.
    b.  **Lookup Candidate:** Query the hash table for $M_{cand}$. Let the result be cell $a$.
    c.  **Verify Geometric Adjacency:**
        i.  If $a$ exists:
            1.  Get the geometric limit of cell $c$ in the direction of face $f$. Let this be $Limit_c$.
            2.  Get the geometric limit of cell $a$ in the direction of face $f_{adj}$. Let this be $Limit_a$.
            3.  If $Limit_c == Limit_a$, the cells are truly face-adjacent. Return $a$.
            4.  Else, the cells are not adjacent (likely separated by other cells due to irregular subdivision). Increment $k = k + 1$ and continue loop.
        ii. If $a$ does not exist, increment $k = k + 1$ and continue loop.
5.  **Fallback to Parent:** If the loop completes without finding a neighbor, the neighbor may be a parent cell. Shift the three least significant bits of the Morton code of $c$ to obtain the parent's Morton code and look it up in the hash table.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_neighbor(cell_c, face_f, hash_table):
    """
    Find the face-adjacent neighbor of cell_c across face_f.
    Handles generalized octree anisotropy.
    """
    level_c = get_level(cell_c)
    coincident_face = get_coincident_face(face_f)
    offset = 1
    
    while offset <= level_c:
        # 1. Compute candidate spatial position
        # Get spatial coordinates of cell_c (e.g., center or min corner)
        coords_c = get_spatial_coordinates(cell_c)
        
        # Add offset in the direction of face_f
        # The direction vector depends on face_f (e.g., +x, -y, etc.)
        direction = get_face_direction(face_f)
        candidate_coords = coords_c + (direction * offset)
        
        # 2. Convert to Morton Code
        candidate_morton = spatial_to_morton(candidate_coords)
        
        # 3. Lookup in Hash Table
        cell_a = hash_table.get(candidate_morton)
        
        if cell_a is not None:
            # 4. Verify Geometric Adjacency
            # Get geometric limit of cell_c in the direction of face_f
            limit_c = get_cell_limit_by_face(cell_c, face_f)
            
            # Get geometric limit of cell_a in the direction of coincident_face
            limit_a = get_cell_limit_by_face(cell_a, coincident_face)
            
            # Check if limits match (ensures true face adjacency)
            if limit_c == limit_a:
                return cell_a
            else:
                # Not adjacent, try larger offset
                offset += 1
        else:
            # Cell not found at this offset, try larger offset
            offset += 1
            
    # 5. Fallback: Check parent
    # Shift 3 least significant bits to get parent Morton code
    parent_morton = cell_c.morton_code >> 3
    parent_cell = hash_table.get(parent_morton)
    
    if parent_cell is not None:
        # Verify parent adjacency if necessary, though parent is typically the neighbor
        # if no direct child neighbor exists at the current level
        return parent_cell
        
    return None
```

**Parameters:**
- **Offset Limit:** The search offset is limited by the current cell's level $L_c$. Anisotropic subdivision usually occurs at the deepest levels, so the offset will not exceed the current depth [S1].

**Edge Cases and Pitfalls:**
- **Irregular Subdivision:** Standard Morton code arithmetic fails because cells are not uniformly subdivided. The geometric limit check is critical to distinguish between cells that are close in Morton space but not geometrically adjacent [S1].
- **Anisotropy:** Cells may be subdivided in only one or two axial directions. The spatial coordinate conversion must account for the actual volume occupied by the cell, not just a regular grid assumption [S1].
- **Parent Lookup:** If no direct neighbor is found at the current level, the neighbor is likely the parent. The algorithm must handle the bit-shifting correctly to retrieve the parent's Morton code [S1].

## Subtopics

### Generalized Octree with Morton Codes
A space subdivision data structure where octree cells do not have to be subdivided in all three main axial directions; they can be subdivided in only one or two axial directions to adapt to anisotropic volumetric data [S1]. It sets itself apart from regular octrees by allowing partial subdivision, requiring spatial coordinates to complement Morton codes for neighbor access [S1]. Choose this when dealing with anisotropic volumetric data where uniform subdivision is inefficient. It builds on [Neighbor Cell Access via Hash Table and Morton Code](6715-neighbor-cell-access-via-hash-table-and-morton-code.md) by providing the underlying data structure that necessitates the geometric verification in the neighbor algorithm.

## Operating characteristics
- **Time Complexity:** Almost constant time $O(1)$ for hash table lookups. The neighbor search loop runs at most $L_c$ times, where $L_c$ is the cell level. In practice, the offset is small, making it effectively constant [S1].
- **Memory Complexity:** Requires storing spatial coordinates for each cell in addition to the Morton code and hash table overhead [S1].
- **Strengths:** Handles irregular and anisotropic octrees efficiently; ensures geometric correctness of adjacency [S1].
- **Limitations:** Requires storing extra spatial metadata; hash table collisions or resizing can impact performance [S1].
- **When to Use:** Surface extraction from anisotropic volumetric data where standard octree neighbor finding fails due to irregular subdivision [S1].

## Background
Originated in the context of adaptive surface extraction from anisotropic volumetric data. The method was developed to handle generalized octrees where cells are not uniformly subdivided, requiring a robust neighbor-finding mechanism that relies on geometric limits rather than just Morton code arithmetic [S1].

## Sources
[S1] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
