# Greedy Meshing for Opaque/Undefined Blocks

Also called: Opaque Greedy Meshing, Undefined Block Meshing

## Overview
This method extends standard greedy face merging to handle voxel volumes containing both opaque and undefined (transparent/empty) blocks, ensuring that only exposed faces of opaque materials are meshed while correctly handling visibility against undefined neighbors. It differs from standard [Greedy Meshing](1776-greedy-meshing.md) by explicitly defining visibility logic for "undefined" block types, preventing the emission of internal faces between opaque blocks and undefined space, and ensuring no geometry is generated for undefined blocks themselves. It builds on [Greedy face merging](8935-greedy-face-merging.md).

## How it works
**Inputs:** A 3D voxel grid $V$ of size $D_x \times D_y \times D_z$. Each voxel $V(x,y,z)$ has a material ID. A special ID (e.g., 0 or -1) represents "undefined" or empty space. A function `has_appearance(id)` returns true if the material ID corresponds to a renderable opaque block.
**Outputs:** A list of quads, each defined by position, normal, width, height, material ID, and associated UV/color attributes.

**Data Structures:**
*   **Visibility Mask:** A 3D boolean array `visited[D_x][D_y][D_z]`, initialized to false, tracking which voxel faces have been processed.
*   **2D Slice Mask:** A 2D array `mask[D_t][D_b]` for the current sweep slice, storing material IDs of exposed faces or 0 if not exposed/processed.

**Steps:**
1.  **Initialize:** Set `visited` to all false. Initialize output mesh lists for positions, normals, UVs, and colors.
2.  **Sweep Normals:** Iterate over the three principal axes $n \in \{0, 1, 2\}$ (X, Y, Z). Let $t = (n + 1) \pmod 3$ and $b = (n + 2) \pmod 3$ be the tangent and bitangent axes.
3.  **Sweep Slices:** For each slice index $s$ from $0$ to $D_n - 1$:
    a.  **Construct 2D Mask:** Initialize `mask` of size $D_t \times D_b$ to 0.
    b.  **Visibility Check:** For each cell $(t, b)$ in the slice:
        i.  Let $p = (x,y,z)$ be the voxel coordinates where the $n$-component is $s$.
        ii. Determine the neighbor $p_{neighbor}$ in the direction of the normal $n$. If $n=0$ (X), $p_{neighbor} = (s-1, t, b)$ for positive face or $(s+1, t, b)$ for negative face. *Note: The parent algorithm sweeps in one direction per pass; this variant must check both positive and negative normals for each slice to capture all exposed faces, or run two passes per axis (one for positive, one for negative).*
        iii. **Exposure Logic:** A face at $p$ facing normal $n$ is exposed if:
            *   `has_appearance(V(p))` is true.
            *   AND (`p_{neighbor}` is out of bounds OR `V(p_{neighbor})` is undefined OR `!has_appearance(V(p_{neighbor}))`).
        iv. If exposed, set `mask[t][b] = V(p)`. Else, `mask[t][b] = 0`.
    c.  **Greedy Expansion:** Iterate through `mask` in lexicographic order (row $t$, then column $b$).
        i.  Find the first cell $(t_0, b_0)$ where `mask[t_0][b_0] != 0` and `!visited[p]` (where $p$ is the 3D coordinate corresponding to $(t_0, b_0, s)$).
        ii. **Expand Width ($w$):** Start $w=1$. While $t_0 + w < D_t$, `mask[t_0 + w][b_0] == mask[t_0][b_0]`, and `!visited` for all cells in the column segment, increment $w$.
        iii. **Expand Height ($h$):** Start $h=1$. While $b_0 + h < D_b$:
            *   Check if all cells in the rectangle $[t_0, t_0 + w) \times [b_0, b_0 + h)$ have `mask` value equal to `mask[t_0][b_0]` and are unvisited.
            *   If yes, increment $h$. Else, break.
        iv. **Emit Quad:**
            *   Create a quad with origin at $p$ (adjusted for normal direction), normal $n$, width $w$, height $h$.
            *   Add UVs: Extend mesh's UVs array with 4 sets of `[0.0, 2.0]` coordinates [S1, unofficial].
            *   Add Colors: Extend mesh's colors array with 4 sets of `[0.0, 0.0, 0.0, 1.0]` (or similar default) [S1, unofficial].
            *   *Error Handling:* If `has_appearance` fails for the material ID, assert "No appearance defined for block face" [S1, unofficial].
        v.  **Mark Processed:** Set `visited` to true for all voxel faces corresponding to the rectangle $[t_0, t_0 + w) \times [b_0, b_0 + h)$ in the current slice. Set `mask` cells to 0 to prevent re-processing in this pass.
        vi. Advance scan cursor to $t_0 + w$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh_opaque_undefined(V, D_x, D_y, D_z, has_appearance):
    visited = [[[False for _ in range(D_z)] for _ in range(D_y)] for _ in range(D_x)]
    quads = []
    
    # Directions: 0=X, 1=Y, 2=Z
    # For each axis, we need to check both positive and negative faces
    for n in range(3):
        t = (n + 1) % 3
        b = (n + 2) % 3
        
        # Sweep in positive direction (face normal points in +n direction)
        # Face at s is exposed if V(s) is opaque and V(s-1) is undefined/out-of-bounds
        for s in range(D_n): # D_n is D_x if n=0, etc.
            mask = [[0 for _ in range(D_b)] for _ in range(D_t)]
            for tt in range(D_t):
                for bb in range(D_b):
                    # Get 3D coords
                    p = [0,0,0]
                    p[n] = s
                    p[t] = tt
                    p[b] = bb
                    
                    # Neighbor in negative n direction
                    p_neg = p.copy()
                    p_neg[n] -= 1
                    
                    is_exposed = False
                    if has_appearance(V[p]):
                        if p_neg[n] < 0 or not has_appearance(V[p_neg]):
                            is_exposed = True
                    if is_exposed:
                        mask[tt][bb] = V[p]
            
            # Greedy expansion on mask
            for tt in range(D_t):
                for bb in range(D_b):
                    if mask[tt][bb] != 0 and not visited[p]: # p derived from tt, bb, s
                        w = 1
                        while tt + w < D_t and mask[tt+w][bb] == mask[tt][bb] and not visited[get_p(tt+w, bb, s, n, t, b)]:
                            w += 1
                        
                        h = 1
                        while h <= D_b - bb:
                            # Check rectangle [tt, tt+w) x [bb, bb+h)
                            rect_valid = True
                            for r_tt in range(tt, tt + w):
                                for r_bb in range(bb, bb + h):
                                    if mask[r_tt][r_bb] != mask[tt][bb] or visited[get_p(r_tt, r_bb, s, n, t, b)]:
                                        rect_valid = False
                                        break
                                if not rect_valid: break
                            if rect_valid:
                                h += 1
                            else:
                                break
                        
                        # Emit quad
                        quad = create_quad(p, n, w, h, mask[tt][bb])
                        quad.uvs = [[0.0, 2.0], [0.0, 2.0], [0.0, 2.0], [0.0, 2.0]] # [S1, unofficial]
                        quad.colors = [[0.0, 0.0, 0.0, 1.0], [0.0, 0.0, 0.0, 1.0], [0.0, 0.0, 0.0, 1.0], [0.0, 0.0, 0.0, 1.0]] # [S1, unofficial]
                        quads.append(quad)
                        
                        # Mark visited
                        for r_tt in range(tt, tt + w):
                            for r_bb in range(bb, bb + h):
                                visited[get_p(r_tt, r_bb, s, n, t, b)] = True
                                mask[r_tt][r_bb] = 0
                        
                        tt += w # Advance cursor
                        break # Break inner loop to continue outer scan
                    # Note: Standard greedy meshing advances cursor. The above logic needs careful cursor management.
                    # Simplified: just mark and continue. The 'advance' is handled by the loop structure if we skip processed cells.
                    # Correct implementation:
                    # if mask[tt][bb] != 0 and not visited:
                    #    ... expand ...
                    #    mark visited
                    #    tt += w - 1 (because loop increments)
                    # else:
                    #    continue
```
*Note: The pseudocode above is a conceptual sketch. A robust implementation requires precise cursor advancement to avoid re-scanning processed cells.*

**Parameters:**
*   **Undefined ID:** The integer value representing empty/transparent space. Default: 0.
*   **Boundary Policy:** Voxels outside the grid bounds are treated as undefined [S12, S13, unofficial].

## Subtopics

### Chunk Neighbor Optimization
An optimization where chunks are made aware of their neighboring chunks to remove inner polygons that are never visible, solving the issue where neighboring chunks previously did not know about each other [S2, unofficial].
*   **Definition:** Chunks pass neighbor coordinates to each other during mesh generation to check visibility against adjacent chunk boundaries [S2, unofficial].
*   **Step:** Construct the world cube by cube. Pass the neighbor's coordinates (if one exists) to the current cube. Check against each of the cube's sides as it is created to determine visibility [S2, unofficial].
*   **Pitfall:** Unity restricts a mesh to approximately 65,000 vertices. A Minecraft-style chunk of 65,536 cubes exceeds this limit, so a single chunk cannot be made. The solution is to recombine meshes by stacking multiple chunks on top of each other [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(D_x D_y D_z)$ per axis pass, with a constant factor depending on the average quad size. The greedy expansion is linear in the number of voxels if implemented with efficient visited tracking.
*   **Memory Complexity:** $O(D_x D_y D_z)$ for the visited array and $O(N_{quads})$ for the output mesh.
*   **Strengths:** Significantly reduces triangle count compared to per-voxel meshing. Handles undefined blocks correctly by culling internal faces.
*   **Limitations:** Does not handle transparency blending (only opaque blocks are meshed). Performance degrades if the volume has high surface complexity (many small exposed faces).
*   **When to use:** When rendering large voxel worlds with opaque blocks and empty space, where draw call reduction is critical.

## Background
This method is a variant of greedy meshing adapted for voxel engines like Minecraft, where "undefined" blocks (air) are distinct from opaque blocks. The concept of chunk neighbor optimization was developed to address visibility issues between adjacent chunks in static world generation [S2, unofficial]. The specific handling of UVs and colors for opaque blocks is documented in the Infinigen project [S1, unofficial].

## Sources
[S1] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S2] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
