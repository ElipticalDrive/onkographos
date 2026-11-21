# TileTree Rendering

Also called: TileTree

## Overview
TileTree Rendering is a method for storing and accessing high-resolution texture data on octree-structured volumes, enabling seamless interpolation and adaptive resolution [S1]. It differs from standard octree texturing by storing separate tile maps for each face of the octree leaves and using a weighted blending scheme based on surface normal direction to eliminate visible seams at tile boundaries [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D position $p$ and a surface normal $IN.Nrm$.
*   **Output:** A 4D color value (RGBA) representing the texture at $p$.

**Data Structures:**
Each octree leaf node stores:
1.  A 6-bit bit vector marking tile presence on each of the 6 faces (positive/negative X, Y, Z) [S1].
2.  Two 16-bit integers storing the coordinates of the top-left corner of the tiles within the global tile map [S1].
3.  An 8-bit integer storing the resolution of the tiles $tileres$ [S1].

**Algorithm Steps:**
1.  **Octree Lookup:** Perform an octree lookup at position $p$ to retrieve the leaf node data [S1].
2.  **Decode Face Presence:** Decode the 6-bit presence vector into two 3D vectors: $face_p$ (positive X, Y, Z) and $face_n$ (negative X, Y, Z), where components are 1 if the tile is present and 0 otherwise [S1].
3.  **Local Coordinates:** Compute local coordinates within the node by scaling $p$ by the node's level size ($lvlsize$) and taking the fractional part:
    $$ local = \text{frac}(p \times lvlsize) $$
    [S1]
4.  **UVW Alignment:** Decode the tile resolution $tileres$ from the leaf. Compute UVW coordinates aligned on leaf boundaries to ensure seamless interpolation:
    $$ uvw = local \times \frac{tileres - 1.0}{tileres} + \frac{0.5}{tileres} $$
    [S1]
5.  **Face Selection:** Normalize the input normal $IN.Nrm$ to get $nrm$. Compute view vectors for positive and negative faces:
    $$ v_p = face_p \times nrm $$
    $$ v_n = face_n \times (-nrm) $$
    [S1]
    Select the active face ID based on the view direction. If $v_p > 0$, use $id_p = (X_P, Y_P, Z_P)$; otherwise, if $v_n > 0$, use $id_n = (X_N, Y_N, Z_N)$; otherwise, the ID is -1. Store the result in $faceid$ [S1].
6.  **Tile Access:** Initialize colors $clr0, clr1, clr2$ to 0. Access tile data for present tiles using the `tileLkup` function:
    *   If $faceid.x > -1$, $clr0 = \text{tileLkup}(leaf, faceid.x, uvw.yz)$.
    *   If $faceid.y > -1$, $clr1 = \text{tileLkup}(leaf, faceid.y, uvw.xz)$.
    *   If $faceid.z > -1$, $clr2 = \text{tileLkup}(leaf, faceid.z, uvw.xy)$.
    [S1]
7.  **Sample Usage Vector:** Compute the alpha usage vector $\alpha_{xyz}$ from the alpha channels (w component) of the sampled colors, adding a small epsilon ($10^{-6}$) for robustness:
    $$ \alpha_{xyz} = 10^{-6} + \text{float3}(clr0.w, clr1.w, clr2.w) $$
    [S1]
8.  **Damping Factor:** Compute the damping factor for seamless interpolation. Calculate $anrm = |nrm|$. Compute:
    $$ damp = \frac{anrm}{\max(anrm.x, \max(anrm.y, anrm.z))} $$
    Multiply $\alpha_{xyz}$ by $damp$ [S1].
9.  **Weight Computation:** Compute weights $w$ for blending. Calculate $inv = 1 - \alpha_{xyz}$. Compute:
    $$ w = \alpha_{xyz} \times inv.yzx \times inv.zxy $$
    [S1]
10. **Final Color:** Compute the final color by the weighted sum of $clr0, clr1, clr2$ using weights $w.x, w.y, w.z$, divided by the dot product of $w$ and 1:
    $$ \text{return } \frac{clr0 \times w.x + clr1 \times w.y + clr2 \times w.z}{\text{dot}(w, 1)} $$
    [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tileTreeLookup(p, IN_Nrm, octree_lookup, decode_pos_face_presence, decode_neg_face_presence, decode_tile_resolution, tileLkup, lvlsize):
    # 1. Octree Lookup
    leaf = octree_lookup(p)
    
    # 2. Decode Face Presence
    face_p = decode_pos_face_presence(leaf)
    face_n = decode_neg_face_presence(leaf)
    
    # 3. Local Coordinates
    local = frac(p * lvlsize)
    
    # 4. UVW Alignment
    tileres = decode_tile_resolution(leaf)
    uvw = local * ((tileres - 1.0) / tileres) + (0.5 / tileres)
    
    # 5. Face Selection
    nrm = normalize(IN_Nrm)
    v_p = face_p * nrm
    v_n = face_n * (-nrm)
    
    id_p = float3(1.0, 1.0, 1.0) # X_P, Y_P, Z_P indicators (assuming 1 if present in face_p)
    id_n = float3(1.0, 1.0, 1.0) # X_N, Y_N, Z_N indicators (assuming 1 if present in face_n)
    
    # Note: The source uses face_p and face_n as masks. 
    # The logic implies id_p and id_n are indices or masks. 
    # Based on source: "if faceid.x > -1". 
    # We assume face_p and face_n contain 1.0 for present, 0.0 for absent.
    # The source code: float3 id_p = float3(X_P,Y_P,Z_P); 
    # This implies X_P, Y_P, Z_P are specific identifiers or the presence mask itself.
    # Given the context "Select the active face ID... if v_p > 0", 
    # and the subsequent access "tileLkup(leaf, faceid.x, ...)", 
    # faceid.x must be a valid index or flag. 
    # The source code sets faceid = (v_p > 0) ? id_p : -1.
    # If id_p is the presence mask, this logic is slightly ambiguous in isolation,
    # but the standard interpretation in this context is that id_p/n are the presence vectors themselves
    # or indices derived from them. However, the source explicitly defines:
    # float3 id_p = float3(X_P,Y_P,Z_P);
    # Let's stick to the source's variable names for the mask logic.
    # Actually, looking at "v_p = face_p * nrm", face_p is the presence mask.
    # The selection logic checks if the dot product is positive.
    # The source code: float3 faceid = (v_p > 0) ? id_p : -1;
    # This suggests id_p is a vector of indices or flags. 
    # For implementation, we treat id_p and id_n as the presence masks (1.0/0.0) 
    # or specific indices if the API requires. 
    # Given "tileLkup(leaf, faceid.x, ...)", faceid.x is likely an index.
    # However, the source does not define X_P, Y_P, Z_P as variables but as placeholders for indices.
    # We will assume faceid.x/y/z are the indices of the tiles on the respective faces.
    # The source code snippet is:
    # float3 id_p = float3(X_P,Y_P,Z_P);
    # This implies X_P, Y_P, Z_P are constants or variables representing the tile indices for the positive faces.
    # Since the leaf stores "coordinates of the top left corner", the indices are likely derived or stored.
    # For the purpose of this article, we follow the source's structure exactly.
    
    faceid = float3(-1.0, -1.0, -1.0)
    if v_p.x > 0: faceid.x = X_P # Placeholder for actual index logic from source
    if v_p.y > 0: faceid.y = Y_P
    if v_p.z > 0: faceid.z = Z_P
    
    if v_n.x > 0: faceid.x = X_N
    if v_n.y > 0: faceid.y = Y_N
    if v_n.z > 0: faceid.z = Z_N
    
    # Re-evaluating source: "float3 faceid = (v_p > 0) ? id_p : -1; faceid = (v_n > 0) ? id_n : faceid;"
    # This is component-wise or scalar? The source uses float3 comparisons.
    # In GLSL/HLSL, (v_p > 0) is component-wise.
    # So faceid is a vector where each component is either the corresponding id_p/n or -1.
    
    faceid = select(v_p > 0, id_p, float3(-1,-1,-1))
    faceid = select(v_n > 0, id_n, faceid)
    
    # 6. Tile Access
    clr0 = float4(0,0,0,0)
    clr1 = float4(0,0,0,0)
    clr2 = float4(0,0,0,0)
    
    if faceid.x > -1: clr0 = tileLkup(leaf, faceid.x, uvw.yz)
    if faceid.y > -1: clr1 = tileLkup(leaf, faceid.y, uvw.xz)
    if faceid.z > -1: clr2 = tileLkup(leaf, faceid.z, uvw.xy)
    
    # 7. Sample Usage Vector
    alpha_xyz = float3(clr0.w, clr1.w, clr2.w) + 1e-6
    
    # 8. Damping Factor
    anrm = abs(nrm)
    max_anrm = max(anrm.x, max(anrm.y, anrm.z))
    damp = anrm / max_anrm
    alpha_xyz = alpha_xyz * damp
    
    # 9. Weight Computation
    inv = 1.0 - alpha_xyz
    w = alpha_xyz * inv.yzx * inv.zxy
    
    # 10. Final Color
    return (clr0 * w.x + clr1 * w.y + clr2 * w.z) / dot(w, 1)
```
*[G] The variables X_P, Y_P, Z_P, X_N, Y_N, Z_N in the source pseudocode represent the specific tile indices or identifiers for the positive/negative faces. In a concrete implementation, these would be extracted from the leaf data or derived from the face presence bits. The `select` operation is assumed to be component-wise.*

## Subtopics

### TileTree Shader Lookup
This variant refers to the core shader logic described in the main method, specifically focusing on the octree traversal, face selection, UV calculation, and tile access steps [S1]. It emphasizes the extraction of sample usage flags (alpha) from tile colors with epsilon addition for robustness [S1].

### MIP-mapping for TileTrees
Due to the tile resolution being $2k+1$, standard MIP-mapping cannot be directly applied to the tile map [S1]. This variant uses a strategy where a separate tile map is computed for each resolution level [S1]. One tile coordinate per level is stored in the octree leaves [S1]. During rendering, the appropriate MIP level is computed and the color is fetched from the corresponding tile map; two levels may be accessed for linear interpolation between MIP levels [S1]. Additionally, the tree itself may be MIP-mapped in cases of extreme undersampling [S1].

## Operating characteristics
**Time and Memory Complexity:**
The method requires an octree lookup and a fixed number of tile lookups (up to 3) per sample. The memory footprint is efficient; for example, an Armadillo model textured at $1024^3$ resolution fits in 11.4 MB, and a Dragon model fits in 11.3 MB [S1].

**Strengths and Limitations:**
The primary strength is seamless interpolation and adaptive resolution, eliminating visible seams at tile boundaries [S1]. The limitation is the complexity of handling non-power-of-two tile resolutions ($2k+1$) for MIP-mapping, requiring separate tile maps per level [S1].

**When to Use:**
Use when high-resolution texture storage is required on sparse volumes with the need for seamless interpolation and adaptive resolution [S1].

## Background
TileTree Rendering was introduced in the paper "TileTrees" [S1]. It builds on hardware implementations of octree traversal [S1]. The method allows for interactive painting and conversion between TileTrees and other texture representations [S1].

## Sources
[S1] TileTrees, https://doi.org/10.1145/1230100.1230104, fetched 2026-10-26
