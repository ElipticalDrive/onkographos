# Greedy Meshing for Opaque/Undefined Blocks


## Overview
This method extends standard greedy face merging to handle voxel volumes containing opaque and undefined (empty/transparent) blocks, ensuring only exposed faces of opaque materials are meshed [G]. It differs from standard [Greedy Meshing](3519-greedy-meshing.md) by explicitly defining visibility logic for undefined block types, preventing internal face emission between opaque blocks and undefined space [G]. It builds on [Greedy face merging](8935-greedy-face-merging.md).

## How it works
**Inputs:** A 3D voxel grid $V$ of size $D_x \times D_y \times D_z$. Each voxel $V(x,y,z)$ holds a material ID. An undefined/empty ID (typically $0$) represents non-opaque space. A predicate `has_appearance(id)` returns true for renderable opaque materials.
**Outputs:** A list of quads, each defined by position, normal, width, height, material ID, and associated UV/color attributes.

**Data Structures:**
*   **Visited Array:** A 3D boolean array `visited[D_x][D_y][D_z]`, initialized to false, tracking which voxel faces have been processed.
*   **2D Slice Mask:** A 2D integer array `mask[D_t][D_b]` for the current sweep slice, storing material IDs of exposed faces or $0$ if not exposed/processed.

**Steps:**
1.  **Initialize:** Set `visited` to all false. Initialize output mesh lists for positions, normals, UVs, and colors.
2.  **Sweep Normals:** Iterate over the three principal axes $n \in \{0, 1, 2\}$ (X, Y, Z). Let $t = (n + 1) \pmod 3$ and $b = (n + 2) \pmod 3$ be the tangent and bitangent axes.
3.  **Sweep Slices:** For each slice index $s$ from $0$ to $D_n - 1$:
    a.  **Construct 2D Mask:** Initialize `mask` of size $D_t \times D_b$ to $0$.
    b.  **Visibility Check:** For each cell $(t, b)$ in the slice:
        i.  Let $p = (x,y,z)$ be the voxel coordinates where the $n$-component is $s$.
        ii. Determine the neighbor $p_{neighbor}$ in both directions along $n$ ($p \pm \hat{n}$). A face at $p$ facing normal $n$ is exposed if `has_appearance(V[p])` is true AND (`p_{neighbor}` is out of bounds OR `V[p_{neighbor}]` is undefined OR `!has_appearance(V[p_{neighbor}])`) [G]. Boundary handling treats out-of-bounds neighbors as undefined [G].
        iii. If exposed, set `mask[t][b] = V[p]`. Else, `mask[t][b] = 0`.
    c.  **Greedy Expansion:** Iterate through `mask` in lexicographic order (row $t$, then column $b$).
        i.  Find the first cell $(t_0, b_0)$ where `mask[t_0][b_0] != 0` and `!visited[p]` (where $p$ is the 3D coordinate corresponding to $(t_0, b_0, s)$).
        ii. **Expand Width ($w$):** Start $w=1$. While $t_0 + w < D_t$, `mask[t_0 + w][b_0] == mask[t_0][b_0]`, and the corresponding voxel is unvisited, increment $w$.
        iii. **Expand Height ($h$):** Start $h=1$. While $b_0 + h < D_b$:
            *   Check if all cells in the rectangle $[t_0, t_0 + w) \times [b_0, b_0 + h)$ have `mask` value equal to `mask[t_0][b_0]` and are unvisited.
            *   If yes, increment $h$. Else, break.
        iv. **Emit Quad:** Create a quad with origin at $(t_0, b_0, s)$, normal $n$, width $w$, height $h$. Extend the mesh's UVs array with 4 sets of `[0.0; 2]` UV coordinates. Extend the mesh's colors array with 4 sets of `[0.0; 4]` color coordinates [S1, unofficial]. If `has_appearance` fails for the material ID, assert failure with the message "No appearance defined for block face" [S1, unofficial].
        v.  **Mark Processed:** Set `visited` to true for all voxel faces corresponding to the rectangle $[t_0, t_0 + w) \times [b_0, b_0 + h)$ in the current slice. Set `mask` cells to $0$ to prevent re-processing in this pass.
        vi. Advance scan cursor to $t_0 + w$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_mesh_opaque_undefined(V, dims, has_appearance):
    Dx, Dy, Dz = dims
    visited = [[[False]*Dz for _ in range(Dy)] for _ in range(Dx)]
    quads = []
    axes = [(0,1,2), (1,2,0), (2,0,1)] # (n, t, b)
    for n, t, b in axes:
        Dn, Dt, Db = dims[n], dims[t], dims[b]
        for s in range(Dn):
            mask = [[0]*Db for _ in range(Dt)]
            for tt in range(Dt):
                for bb in range(Db):
                    p = [0,0,0]; p[n]=s; p[t]=tt; p[b]=bb
                    exposed = False
                    for dir in [-1, 1]:
                        p_nb = [p[0], p[1], p[2]]; p_nb[n] += dir
                        if not (0 <= p_nb[n] < Dn):
                            if has_appearance(V[p]):
                                exposed = True; break
                        else:
                            nb_val = V[p_nb[0]][p_nb[1]][p_nb[2]]
                            if has_appearance(V[p]) and (nb_val == 0 or not has_appearance(nb_val)):
                                exposed = True; break
                    if exposed:
                        mask[tt][bb] = V[p]
            for tt in range(Dt):
                for bb in range(Db):
                    if mask[tt][bb] == 0: continue
                    p = [0,0,0]; p[n]=s; p[t]=tt; p[b]=bb
                    if visited[p[0]][p[1]][p[2]]: continue
                    w = 1
                    while tt + w < Dt and mask[tt+w][bb] == mask[tt][bb]:
                        p_check = [0,0,0]; p_check[n]=s; p_check[t]=tt+w; p_check[b]=bb
                        if visited[p_check[0]][p_check[1]][p_check[2]]: break
                        w += 1
                    h = 1
                    while bb + h < Db:
                        rect_ok = True
                        for r_tt in range(tt, tt+w):
                            for r_bb in range(bb, bb+h):
                                p_check = [0,0,0]; p_check[n]=s; p_check[t]=r_tt; p_check[b]=r_bb
                                if mask[r_tt][r_bb] != mask[tt][bb] or visited[p_check[0]][p_check[1]][p_check[2]]:
                                    rect_ok = False; break
                            if not rect_ok: break
                        if rect_ok: h += 1
                        else: break
                    quad_pos = [0,0,0]; quad_pos[n]=s; quad_pos[t]=tt; quad_pos[b]=bb
                    quads.append({"pos": quad_pos, "normal": n, "w": w, "h": h, "mat": mask[tt][bb]})
                    quads[-1]["uvs"] = [[0.0]*2 for _ in range(4)]
                    quads[-1]["colors"] = [[0.0]*4 for _ in range(4)]
                    for r_tt in range(tt, tt+w):
                        for r_bb in range(bb, bb+h):
                            p_mark = [0,0,0]; p_mark[n]=s; p_mark[t]=r_tt; p_mark[b]=r_bb
                            visited[p_mark[0]][p_mark[1]][p_mark[2]] = True
                            mask[r_tt][r_bb] = 0
    return quads
```

**Parameters:**
*   **Undefined ID:** The integer value representing empty/transparent space. Default: $0$.
*   **Boundary Policy:** Voxels outside the grid bounds are treated as undefined [G].

## Subtopics

### Chunk Neighbor Optimization
An optimization where chunks are made aware of their neighboring chunks to remove inner polygons that are never visible, solving the issue where neighboring chunks previously did not know about each other [S2, unofficial].
*   **Definition:** Chunks pass neighbor coordinates to each other during mesh generation to check visibility against adjacent chunk boundaries [S2, unofficial].
*   **Step:** Construct the world cube by cube. Pass the neighbor's coordinates (if one exists) to the current cube. Check against each of the cube's sides as it is created to determine visibility [S2, unofficial].
*   **Pitfall:** Unity restricts a mesh to approximately 65,000 vertices. A Minecraft-style chunk of 65,536 cubes exceeds this limit, so a single chunk cannot be made. The solution is to recombine meshes by stacking multiple chunks on top of each other [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(D_x D_y D_z)$ per axis pass, with a constant factor depending on the average quad size. The greedy expansion is linear in the number of voxels if implemented with efficient visited tracking [G].
*   **Memory Complexity:** $O(D_x D_y D_z)$ for the visited array and $O(N_{quads})$ for the output mesh [G].
*   **Strengths:** Significantly reduces triangle count compared to per-voxel meshing. Handles undefined blocks correctly by culling internal faces [G].
*   **Limitations:** Does not handle transparency blending (only opaque blocks are meshed). Performance degrades if the volume has high surface complexity (many small exposed faces) [G].
*   **When to use:** When rendering large voxel worlds with opaque blocks and empty space, where draw call reduction is critical [G].

## Background
This method is a variant of greedy meshing adapted for voxel engines like Minecraft, where "undefined" blocks (air) are distinct from opaque blocks [G]. The concept of chunk neighbor optimization was developed to address visibility issues between adjacent chunks in static world generation [S2, unofficial]. The specific handling of UVs and colors for opaque blocks is documented in the Infinigen project [S1, unofficial].

## Sources
[S1] jameshiew/infinigen, https://github.com/jameshiew/infinigen, fetched 2026-10-27 (unofficial)
[S2] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
