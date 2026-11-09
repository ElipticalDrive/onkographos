# Octree Neighbor Search

Also called: Get Neighbour Algorithm

## Overview
An algorithm to find the adjacent cell sharing a specific face with a given cell in a generalized octree, handling cases where the neighbor might be at a different subdivision level [S1]. It differs from sibling methods like [SVO Pathfinder with Lazy Theta* and LOS Recovery](7910-svo-pathfinder-with-lazy-theta-and-los-recovery.md) by focusing on spatial adjacency in hierarchical structures rather than pathfinding in sparse voxel grids.

## How it works
**Inputs:** A cell $c$ and a face $f$.
**Outputs:** An adjacent cell $a$ through face $f$, or null if no neighbor exists.

**Data Structures:**
*   **Cell:** Identified by a Morton code (Z-order curve). Contains implicit spatial coordinates derived from the code and level.
*   **Face:** One of the 6 directions (±X, ±Y, ±Z).
*   **Level:** The subdivision depth of the cell.

**Helper Definitions:**
*   `GetLevel(c)`: Returns the subdivision level of cell $c$.
*   `GetCoincidentFace(f)`: Returns the face on the opposite side of the neighbor relative to $f$. For example, if $f$ is +X, the coincident face is -X.
*   `GetAdjacentCellByFace(c, f, offset)`: Returns a candidate cell. It obtains the spatial coordinates of $c$, adds an `offset` in the direction of $f$, and converts the result to a Morton code. The offset represents the distance in voxel units at the finest resolution [S1].
*   `Exists(a)`: Checks if a cell with Morton code $a$ exists in the octree.
*   `GetCellLimitByFace(c, f)`: Returns the geometric boundary coordinate of cell $c$ in the direction of face $f$ [S1].

**Algorithm Steps:**
1.  Let $L \leftarrow \text{GetLevel}(c)$.
2.  Let $f_{\text{coinc}} \leftarrow \text{GetCoincidentFace}(f)$.
3.  Let $\text{offset} \leftarrow 1$.
4.  While $\text{offset} \le L$:
    a.  $a \leftarrow \text{GetAdjacentCellByFace}(c, f, \text{offset})$.
    b.  If $\text{Exists}(a)$:
        i.  $L_c \leftarrow \text{GetCellLimitByFace}(c, f)$.
        ii. $L_a \leftarrow \text{GetCellLimitByFace}(a, f_{\text{coinc}})$.
        iii. If $L_c == L_a$:
            Return $a$.
        iv. Else:
            $\text{offset} \leftarrow \text{offset} + 1$.
    c.  Else:
        $\text{offset} \leftarrow \text{offset} + 1$.
5.  Return null.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_neighbor(c, f):
    level = get_level(c)
    coincident_face = get_coincident_face(f)
    offset = 1
    
    while offset <= level:
        # Get candidate neighbor by shifting spatial coordinates
        # and converting to Morton code.
        # If no neighbor exists at this offset, the code generation
        # might return an invalid code or a code for a non-existent cell.
        candidate = get_adjacent_cell_by_face(c, f, offset)
        
        if exists(candidate):
            # Get geometric limits to verify adjacency
            limit_c = get_cell_limit_by_face(c, f)
            limit_a = get_cell_limit_by_face(candidate, coincident_face)
            
            if limit_c == limit_a:
                return candidate
            else:
                offset += 1
        else:
            offset += 1
            
    return None
```

**Edge Cases and Pitfalls:**
*   **Anisotropic Subdivision:** The algorithm handles irregular subdivision depths. The offset is bounded by the current cell's level, as anisotropic subdivision usually occurs at deeper levels [S1].
*   **Parent Search:** If no neighbor is found at the current level or shallower offsets, the method may look up to the parent cell by shifting the three least significant bits of the Morton code [S1]. This is implied by the offset logic but not explicitly detailed in the loop structure.
*   **Precision:** Geometric limits must be compared with exact equality. Floating-point errors in limit calculation could cause false negatives.

## Subtopics

### Hong Implicit Neighbor Octree
Generates fragments corresponding to rays intersecting the cell. This approach is used for efficient classification of cells and is stored in a footprint table [S2, unofficial]. It relies on the min-max octree to implicitly give the correct order between sub-volumes.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the level of the cell, as the offset loop runs up to $L$ times.
*   **Memory Complexity:** Constant, assuming Morton code operations and existence checks are $O(1)$ or $O(\log N)$ depending on the storage structure.
*   **Strengths:** Handles generalized octrees with anisotropic subdivision. Does not require regular grid structures.
*   **Limitations:** Requires geometric limit calculations. Performance depends on the efficiency of `Exists` and `GetCellLimitByFace`.

## Background
The algorithm was described by Ricardo Uribe Lobello, Florence Denis, and Florent Dupont in the context of adaptive surface extraction from anisotropic volumetric data [S1]. It leverages spatial coordinates contained within cells to access neighboring cells even if the octree has not been divided regularly.

## Sources
[S1] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
