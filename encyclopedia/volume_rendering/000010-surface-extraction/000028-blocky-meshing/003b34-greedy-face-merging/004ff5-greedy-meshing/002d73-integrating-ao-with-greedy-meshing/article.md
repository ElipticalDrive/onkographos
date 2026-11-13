# Integrating AO with Greedy Meshing

Also called: AO-aware Greedy Meshing, Greedy Meshing with AO-Aware Merge Rule, Greedy Meshing with Ambient Occlusion

## Overview
Integrating Ambient Occlusion (AO) with Greedy Meshing resolves the conflict between merging coplanar voxel faces into large quads and the natural variation of occlusion values across individual block boundaries [S2, unofficial]. Unlike standard greedy meshing which prioritizes minimal polygon count, this method enforces strict visual continuity of lighting by ensuring that the interpolated shading on a merged quad accurately represents the occlusion of every unit face it replaces [S1, unofficial]. It achieves this through specific merge rules, data packing, or shader-based reconstruction strategies [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   A 3D voxel grid with exposed faces.
*   Per-vertex AO values for each unit face. The neighbor-sampling method typically produces four discrete levels per vertex [S2, unofficial].
*   Material type and facing direction for each face [S1, unofficial].

**Outputs:**
*   A mesh of merged quads where vertex attributes (AO, material, normal) are consistent across the entire quad surface [S1, unofficial].

### Data Structures
To minimize memory bandwidth during procedural mesh generation, AO data is packed efficiently [S2, unofficial]:
*   **AO Storage:** Since AO values are discrete (4 levels), each vertex AO requires only 2 bits [S2, unofficial].
*   **Quad Packing:** All four vertices of a quad are stored in 8 bits (1 byte) total [S2, unofficial].
*   **Vertex Structure:** This 1-byte AO value is packed alongside face normal indicators and texture IDs into a single 32-bit unsigned integer per vertex [S2, unofficial].

### Steps
1.  **Define AO-Aware Merge Rule:** Before baking vertex AO, define the rule that ensures the bilinear interpolation of AO across a merged quad equals the AO values of every unit face it replaces [S1, unofficial].
2.  **Evaluate Merge Criteria:** For adjacent coplanar faces, check if they can be merged. A face is eligible for merging if:
    *   It shares the same material type [S1, unofficial].
    *   It has the same facing direction [S1, unofficial].
    *   It has an identical four-corner AO tuple [S1, unofficial].
3.  **Execute Merge (Restricted Strategy):** If the criteria are met, merge the faces into a single quad. This "Restricted Merging" strategy preserves accurate vertex-based shading at the cost of a slightly higher polygon count compared to unrestricted merging [S2, unofficial].
4.  **Alternative: Texture Atlas Baking:** If vertex-based merging is too restrictive, generate a low-resolution texture atlas containing pre-computed AO gradients. Apply UVs from the merged quad across the continuous surface to sample these gradients [S2, unofficial].
5.  **Alternative: Fragment-Shader Reconstruction:** Pass unmerged face bounds as custom vertex attributes or instanced data. The fragment shader calculates edge proximity and distance-to-corner shading analytically, allowing geometry to remain unmerged while shading remains correct [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_ao_consistent(face_a, face_b):
    """
    Check if two adjacent faces can be merged based on AO consistency.
    Returns True if materials, directions, and AO tuples match.
    """
    if face_a.material != face_b.material:
        return False
    if face_a.direction != face_b.direction:
        return False
    
    # Get the 4-corner AO tuple for each face
    # AO values are 2-bit integers (0-3)
    tuple_a = get_four_corner_ao_tuple(face_a)
    tuple_b = get_four_corner_ao_tuple(face_b)
    
    return tuple_a == tuple_b

def get_four_corner_ao_tuple(face):
    """
    Retrieve the AO values for the four corners of a unit face.
    Assumes AO is stored per-vertex in the voxel grid.
    """
    # Order: top-left, top-right, bottom-right, bottom-left
    # Indices depend on face orientation (x, y, or z axis)
    v0 = face.vertices[0].ao
    v1 = face.vertices[1].ao
    v2 = face.vertices[2].ao
    v3 = face.vertices[3].ao
    return (v0, v1, v2, v3)

def pack_vertex_data(normal, material_id, ao_byte):
    """
    Pack vertex data into a 32-bit unsigned integer.
    ao_byte contains 4 x 2-bit AO values (8 bits total).
    """
    # Implementation details for packing depend on specific bit-field layout
    # This is a placeholder for the packing logic described in [S2, unofficial]
    packed = (normal << 24) | (material_id << 16) | ao_byte
    return packed
```

### Parameters and Edge Cases
*   **AO Levels:** The source specifies four discrete levels per vertex [S2, unofficial].
*   **Precision:** Using 2-bit storage for AO prevents precision issues associated with floating-point colors but limits shading granularity [S2, unofficial].
*   **Halo Bricks:** When building volume meshes, a one-cell halo is used across brick seams for culling and AO calculation. A halo brick arrival makes the mesh stale, triggering a re-mesh to drop now-occluded seams [S1, unofficial].
*   **Visual Continuity:** The primary pitfall is incorrect interpolation. If the merge rule is not defined *before* baking, the bilinear interpolation across the merged quad may not match the original unit face AO values, causing visual artifacts [S1, unofficial].

## Operating characteristics
*   **Polygon Count:** Restricted merging results in a slightly higher polygon count than standard greedy meshing because it prevents merging faces with different AO values [S2, unofficial].
*   **Memory Bandwidth:** Packing AO into 2-bit fields within a 32-bit integer minimizes memory bandwidth during procedural mesh generation [S2, unofficial].
*   **Performance:** The method is CPU-based for surface generation in the cited implementation, with no GPU involvement in the meshing step [S1, unofficial].
*   **Visual Quality:** Ensures visual continuity of lighting across merged geometry, which is the primary advantage over standard greedy meshing [S1, unofficial].

## Background
This method is part of the broader effort to optimize voxel rendering by reducing draw calls while maintaining visual fidelity [S1, unofficial]. The specific implementation details regarding AO-aware merging and data packing are found in the `spall_mesh` module and related discussions on baking dynamic ambient occlusion in procedural voxels [S1, S2, unofficial].

## Sources
[S1] T05 — Visible voxel baseline (spall_mesh, spall_render), https://github.com/deranjer/spall/pull/10, fetched 2026-10-27 (unofficial)
[S2] Baking Dynamic Ambient Occlusion in Procedural Voxels, https://salivity.github.io/game-development/article/baking-dynamic-ambient-occlusion-in-procedural-voxels, fetched 2026-10-29 (unofficial)
