# Voxel Edit Brush Operations

Also called: Place/Carve/Fill/Paint

## Overview
Voxel Edit Brush Operations provide a set of interactive tools (Place, Carve, Fill, Paint) for modifying unbounded 3D voxel grids using world-space spherical or cylindrical selection volumes [S1, unofficial][S2]. The method solves the problem of real-time, view-independent sculpting and material insertion in volumetric data, ensuring consistent behavior across chunk boundaries by addressing voxels via unbounded world-grid coordinates rather than local indices [S1, unofficial]. Unlike parent methods that may rely on sparse structures or offline processing, this approach often utilizes dense 3D textures for mask lookups to enable O(1) per-fragment operations, avoiding the performance penalty of per-fragment analytic SDF evaluations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D voxel grid (often represented as a `VolumeTex` and a `SculptMap` mask texture), a brush center $P$, and a brush radius $r$. For cylindrical operations, a projection plane is also required [S2].
*   **Outputs:** Modified voxel occupancy or material values in the affected region.

**Data Structures:**
*   **SculptMap:** A 3D texture (sampler3D) storing mask information. Typically, the alpha channel ($a$) represents visibility/erasure masks, and the blue channel ($b$) may represent addition/insertion masks [S2].
*   **VolumeTex:** A 3D texture (sampler3D) storing the base volume data (RGB color, Alpha occupancy) [S2].
*   **Brush Parameters:**
    *   $P$: A 4D vector (`float4`). $P.xyz$ is the 3D center of the brush. $P.a$ is the tool radius $r$ [S2].
    *   For cylindrical tools, $P$ may represent a 2D point on a projection plane, with $r$ as the cylinder radius [S2].

**Steps:**

1.  **Coordinate Addressing:** Address voxels using unbounded world-grid coordinates to ensure consistent behavior across chunk boundaries (e.g., 32-cell boundaries) [S1, unofficial].
2.  **Stroke Accumulation:** Accumulate edits into a stroke across frames. On mouse release, push a single `VoxelEditCommand` to process the batch [S1, unofficial].
3.  **Selection Volume Definition:**
    *   **Spherical (Carve/Fill):** Define a sphere with center $P$ and radius $r$ [S2].
    *   **Cylindrical (Eraser):** Define a projection plane and a 2D point $P$ on it. Transform the cursor to this plane [S2].
4.  **Distance Calculation:**
    *   **Spherical:** Calculate the 3D Euclidean distance $d$ from the voxel center $V$ to $P$:
        $$ d = \text{distance}(P, V) $$
        If $d < r$, the voxel is selected [S2].
    *   **Cylindrical:** Orthogonally project the voxel center $V$ onto the projection plane to get $V_{proj}$. Calculate the 2D Euclidean distance $d$ from $V_{proj}$ to $P$:
        $$ d = \text{length}(V_{proj}.xy - P.xy) $$
        If $d < r$, the voxel is selected [S2].
5.  **Operation Execution:**
    *   **Carve:** If the voxel is selected, remove it (set occupancy to background) [S2].
    *   **Fill:** If the voxel is selected, check if it currently holds a background value. If so, insert material. Do not overwrite existing material [S2].
    *   **Place/Paint:** Modify specific channels of the mask texture (e.g., setting alpha or color channels) based on the selection [S2].

**Pseudocode (Spherical Carve/Fill in Fragment Shader):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs
# v2f.Tex0: Texture coordinates (float2)
# SculptMap: sampler3D (Mask texture)
# VolumeTex: sampler3D (Volume data texture)
# P: float4 (Brush center xyz, radius a)
# PolygonCoord: sampler2D (Projection coordinates if using cylindrical)

struct vertex2fragment {
    float2 Tex0: TEXCOORD0;
};

void main(vertex2fragment v2f,
          uniform sampler3D SculptMap,
          uniform sampler3D VolumeTex,
          uniform float4 P,
          uniform sampler2D PolygonCoord,
          out float4 Col: COLOR0) {
    
    // 1. Sample current state
    float4 mask_voxel = tex3D(SculptMap, v2f.Tex0);
    float4 volume_voxel = tex3D(VolumeTex, v2f.Tex0);
    
    // 2. Get world/projected coordinates
    float4 texCoord = tex2D(PolygonCoord, v2f.Tex0);
    float3 t0 = texCoord.xyz;
    
    // 3. Calculate distance to brush center P
    // P.xyz is center, P.a is radius r
    float D = distance(float4(P.xyz, 1.0f), float4(t0, 1.0f)) / P.a;
    
    // 4. Determine Operation based on Mask Channels
    // Example: Carve if alpha mask is active, Fill if blue mask is active
    
    // Carve Logic (Remove voxel if inside radius and mask indicates carve)
    // Note: Specific channel logic depends on implementation, 
    // but generally involves checking D < 1.0 (normalized distance)
    
    // Fill Logic (Add material if inside radius and voxel is background)
    if (D < 1.0f) {
        // Check if voxel is background (e.g., alpha == 0 or specific channel == 0)
        // Do not overwrite existing material
        if (volume_voxel.r == 0.0f || mask_voxel.a == 0.0f) {
            // Mark for addition (e.g., set blue channel of mask)
            Col.b = 1.0f; 
        } else {
            // Mark for carving/removal
            Col.a = 0.0f; 
        }
    } else {
        // Outside brush, preserve existing state
        Col = mask_voxel;
    }
    
    // 5. Combine with volume data for rendering
    // Col.a = visibility mask
    // Col.rgb = volume color
    Col.rgb = volume_voxel.rgb;
    Col.a = volume_voxel.a * floor(mask_voxel.a);
}
```

**Parameters:**
*   **Brush Radius ($r$):** World-space radius defining the spherical area of influence [S1, unofficial][S2].
*   **Brush Center ($P$):** 3D point specifying the center of the selection volume [S2].
*   **Projection Plane:** Required for cylindrical operations; defines the axis and orientation of the brush [S2].

**Edge Cases and Pitfalls:**
*   **Chunk Boundaries:** Using chunk-local indices causes inconsistent behavior at boundaries. Unbounded world-grid coordinates must be used [S1, unofficial].
*   **Overwriting:** The Fill tool must check current voxel values to avoid overwriting existing material [S2].
*   **Undo/Losslessness:** Snapshots should capture whole chunks, not cell deltas, to allow lossless undo (removing a chunk entirely if it becomes empty) [S1, unofficial].

## Subtopics

### [Voxel Edit Brush Operations — Place/Carve/Fill/Paint, Real-time Voxel Editing, VoxelEdit](12596-voxel-edit-brush-operations-place-carve-fill-paint-real-time.md)
This variant implements real-time voxel carving in a browser-based 3D Gaussian Splatting (3DGS) renderer by injecting a discrete voxel occupancy mask into the fragment shader via a shader patching hook. It solves the problem of crisp, axis-aligned removal of splats using a `discard` instruction, avoiding the performance penalty of per-fragment analytic SDF evaluations or per-splat geometric tests. Unlike parent methods that may rely on sparse structures or offline processing, this approach uses a dense 3D texture sized to the voxel grid, enabling O(1) per-fragment lookup regardless of the number of carved cells.

### FSEraser
A fragment shader that implements the eraser tool for volume sculpting, modifying the alpha channel of a SculptMap texture based on the distance from a 3D point P to the current fragment [S2]. It uses a virtual hand metaphor, detaching the cursor from viewing parameters to allow free rotation and translation [S2].

**Layout:**
*   **Input Struct:** `vertex2fragment` with field `Tex0` (float2, TEXCOORD0) [S2].
*   **Uniforms:**
    *   `SculptMap`: sampler3D, TEXUNIT0 [S2].
    *   `VolumeTex`: sampler3D, TEXUNIT1 [S2].
    *   `P`: float4, C0. `P.xyz` is the 3D point, `P.a` is the tool radius [S2].

**Steps:**
1.  Sample `SculptMap` at `v2f.Tex0`: `Col = tex3D(SculptMap, v2f.Tex0)` [S2].
2.  Calculate normalized 3D Euclidean distance $D$ between $P$ and `texCoord`:
    $$ D = \frac{\text{distance}(\text{float4}(P.xyz, 1), \text{texCoord})}{P.a} $$
    [S2].
3.  If $D < \text{Col}.a$, update `Col.a` to $D$ [S2].

## Operating characteristics
*   **Time Complexity:** O(1) per-fragment lookup for mask access when using dense 3D textures [S1, unofficial]. Stroke accumulation allows batch processing, pushing a single command on release [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of chunks affected by the edit, as only changed chunks are kept in snapshots for undo [S1, unofficial].
*   **Performance:** Avoids per-splat geometric tests or analytic SDF evaluations, offering better performance for real-time editing in dense grids [S1, unofficial].
*   **Strengths:** View-independent, consistent across chunk boundaries, supports lossless undo via chunk snapshots [S1, unofficial].
*   **Limitations:** Dense texture approach may consume significant memory for large grids [S1, unofficial]. Cylindrical tools require additional projection plane calculations [S2].
*   **When to Use:** Real-time voxel sculpting, destructible terrain, interactive volume editing where view independence is critical [S1, unofficial][S2].
*   **When Not to Use:** Extremely large sparse volumes where dense textures are prohibitive, or when analytic SDFs are preferred for smooth surfaces [S1, unofficial].

## Background
The concept of voxel editing brushes originated in 3D sculpting tools that use a virtual hand metaphor to detach the cursor from viewing parameters, allowing free rotation and translation [S2]. The specific implementation of Place/Carve/Fill/Paint operations over unbounded grids with chunk-based undo was developed for real-time voxel engines, such as those used in browser-based 3D Gaussian Splatting renderers [S1, unofficial]. The FSEraser and related tools were described in academic literature on volume sculpting based on geometric tools [S2]. The runtime editing pipeline for spans, cubes, and spheres has been validated against reference implementations like voxlap's `setspans` [S3].

## Sources
[S1] feat(terrain): DDA voxel raycast + editing brushes with undo/redo, https://github.com/drsnuggles8/OloEngineBase/pull/973, fetched 2026-10-25 (unofficial)
[S2] Volume sculpting based on geometric tools, https://doi.org/10.1007/bf03194498, fetched 2026-10-26
[S3] roxlap-cli 0.32.0, https://docs.rs/crate/roxlap-cli/latest, fetched 2026-10-29
