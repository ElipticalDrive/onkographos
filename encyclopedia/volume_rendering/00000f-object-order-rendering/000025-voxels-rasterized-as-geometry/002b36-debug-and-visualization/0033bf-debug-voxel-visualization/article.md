# Debug Voxel Visualization

Also called: DisplayVolume, Voxel Debug Rendering, Voxel Sphere Debug, VoxelMap::visualize

## Overview
Debug Voxel Visualization renders a 3D voxel grid as individual geometric primitives (typically cubes or spheres) for each active voxel by iterating through the voxel data array and drawing a pre-loaded model at every non-transparent position [S2, unofficial]. It solves the problem of visualizing the discrete structure and spatial distribution of voxel data in a scene, distinct from surface extraction methods. It differs from [Axis-Aligned Bounding Box Geometry](1317-axis-aligned-bounding-box-geometry.md) and [Bounding Box Drawing](2733-bounding-box-drawing.md) by rendering individual voxel instances rather than aggregate bounding volumes, allowing for per-voxel inspection [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 1D array of voxel data (typically RGBA bytes), metadata defining the grid dimensions ($w, h, d$), a threshold value for voxel activation, and a pre-loaded geometric model (cube or sphere) [S1, S2, unofficial].
*   **Output:** A set of transformed geometric primitives submitted to the graphics pipeline, positioned at their respective grid coordinates [S2, unofficial].

**Data Structures:**
*   **Voxel Data:** A 1D array of `GLubyte` (RGBA) with size $w \times h \times d \times 4$ bytes [S2, unofficial].
*   **VolumeMetadata:** Parsed from a JSON file (e.g., in `StreamingAssets`), containing dimensions and validation checks ensuring the dimension length is exactly 3 [S1, unofficial].
*   **Voxel Model:** A pre-loaded 3D cube or sphere mesh, centered at the origin [S2, unofficial].

**Parameters:**
*   `intensityThreshold` (float, 0.0–1.0): Voxels with alpha below this value are skipped [S1, unofficial].
*   `dontDraw` (float, default 0.0f): Alternative alpha threshold for skipping voxels [S2, unofficial].
*   `sphereRadius` (float, default 0.5f): Radius of the debug sphere in mm before scaling [S1, unofficial].
*   `voxelOffset` (vec3): Offset applied to align the centered cube model origin to the voxel grid position. Calculated as $(1.0f / voxelmap\_size, 1.0f / voxelmap\_size, 1.0f / voxelmap\_size)$ [S2, unofficial].
*   `maxSpheres` (int, default 5000): Safety cap for spawned debug spheres [S1, unofficial].

**Steps:**
1.  Load metadata JSON and parse into `VolumeMetadata`. Validate that the dimension length is 3 [S1, unofficial].
2.  Iterate through the voxel grid coordinates $x \in [0, w-1]$, $y \in [0, h-1]$, $z \in [0, d-1]$ [S2, unofficial].
3.  Calculate the 1D array index for the current voxel using row-major ordering:
    $$ arrayPos = 4 \times (x + y \times voxelmap\_size + z \times voxelmap\_size \times voxelmap\_size) $$
    where $voxelmap\_size$ is the dimension size (assuming a cubic grid where $w=h=d=size$) [S2, unofficial].
4.  Extract the alpha channel from the RGBA byte at index $arrayPos + 3$ [S2, unofficial].
5.  If the alpha value exceeds the threshold (`intensityThreshold` or `dontDraw`):
    a. Normalize the RGB values by dividing by 255.0f [S2, unofficial].
    b. Set the shader color to the normalized RGB values [S2, unofficial].
    c. Compute the world position using the voxel offset and grid coordinates:
        $$ \mathbf{p}_{world} = \mathbf{voxelOffset} + \text{vC\_to\_wC}(\text{glm::vec3}(x, y, z), voxelmap\_size) $$
        where $\text{vC\_to\_wC}$ converts voxel coordinates to world coordinates based on the grid size [S2, unofficial].
    d. Draw the voxel model at $\mathbf{p}_{world}$ with the set color [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def debug_voxel_visualize(voxel_data: list, metadata: dict, threshold: float, model: Mesh):
    """
    Renders active voxels as geometric primitives.
    
    Args:
        voxel_data: 1D list of RGBA bytes (GLubyte).
        metadata: Dict with 'width', 'height', 'depth', 'size'.
        threshold: Alpha threshold for activation (0.0-1.0).
        model: Pre-loaded geometric mesh (cube/sphere).
    """
    w = metadata['width']
    h = metadata['height']
    d = metadata['depth']
    size = metadata['size'] # Assuming cubic grid for index formula
    
    # Validate dimensions
    if len([w, h, d]) != 3:
        raise ValueError("Metadata must contain exactly 3 dimensions")

    # Calculate voxel offset for centering
    voxel_offset = (1.0 / size, 1.0 / size, 1.0 / size)

    # Iterate through all voxels
    for z in range(d):
        for y in range(h):
            for x in range(w):
                # Calculate 1D array index (row-major, 4 bytes per pixel)
                array_pos = 4 * (x + y * size + z * size * size)
                
                # Extract alpha channel (RGBA layout)
                alpha = voxel_data[array_pos + 3]
                
                # Check threshold
                if alpha > threshold:
                    # Normalize RGB
                    r = voxel_data[array_pos] / 255.0
                    g = voxel_data[array_pos + 1] / 255.0
                    b = voxel_data[array_pos + 2] / 255.0
                    
                    # Compute world position
                    # vC_to_wC converts voxel coords to world coords
                    # Assuming simple scaling for demonstration
                    voxel_coords = (x, y, z)
                    world_pos = (
                        voxel_offset[0] + voxel_coords[0] * (1.0 / size),
                        voxel_offset[1] + voxel_coords[1] * (1.0 / size),
                        voxel_offset[2] + voxel_coords[2] * (1.0 / size)
                    )
                    
                    # Draw model
                    draw_model(model, position=world_pos, color=(r, g, b))
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision issues may arise when converting voxel coordinates to world space if the grid size is very large.
*   **Boundaries:** The `voxelOffset` calculation assumes a specific alignment; incorrect offsets may cause voxels to be misaligned or clipped.
*   **Degenerate Input:** If the voxel data array is empty or dimensions are invalid, the loop will not execute or may throw errors. Validation of metadata is crucial [S1, unofficial].
*   **Performance:** Iterating through all voxels and drawing individual models can be expensive for large grids. The `maxSpheres` cap helps prevent excessive draw calls [S1, unofficial].

## Subtopics

### Voxel Edges
A visual effect that draws outlines of the voxel shapes on the grid with a specified width [S3, unofficial]. It differs from solid voxel rendering by highlighting the boundaries between voxels, useful for inspecting grid structure. Parameter: `width` (float) defining the edge thickness [S3, unofficial].

### Central Slice Visualization
A technique for inspecting dense voxel grids by plotting central slices along X, Y, and Z axes, optionally overlaying hierarchy information from an octree [S4, unofficial]. It differs from full-volume rendering by reducing visual clutter and focusing on specific cross-sections. Steps: Load voxel grid, build octree (`max_depth=8`), plot central slices (`max_depth=5`) with optional red hierarchy overlays [S4, unofficial].

### CSG FieldMutation & Portal X-Ray
A real-time debug overlay highlighting active Constructive Solid Geometry (CSG) field mutations and non-Euclidean portal throats using concentric distance rings [S5, unofficial]. It differs from standard voxel visualization by focusing on SDF-based CSG operations and portal effects. Formula: `max(dTrack, -dSphere)` for CSG union operations [S5, unofficial]. Parameters: `uPortalsA/B` for portal throats [S5, unofficial].

### Live Curtain Sweep
An animated vertical laser scanner curtain used to sweep across the scene for SDF X-Ray visualization [S5, unofficial]. It differs from static visualization by providing a dynamic, animated view of the internal structure. Step: Render an animated vertical curtain sweeping across the viewport [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N = w \times h \times d$, as every voxel must be checked [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the voxel data array [S2, unofficial].
*   **Performance:** Performance degrades linearly with grid size. The `maxSpheres` cap limits draw calls but may truncate visualization for large grids [S1, unofficial].
*   **Strengths:** Simple to implement, provides direct per-voxel visibility, easy to debug individual voxel states [S2, unofficial].
*   **Limitations:** High draw call overhead for large grids, not suitable for real-time visualization of very large volumes without instancing or culling [S1, S2, unofficial].
*   **When to Use:** Debugging voxel data integrity, small-scale voxel worlds, educational purposes [S2, unofficial].
*   **When Not to Use:** Large-scale production rendering, real-time visualization of massive voxel grids [S1, unofficial].

## Background
Debug Voxel Visualization is a fundamental technique in voxel-based applications, originating from the need to inspect discrete volumetric data. It is commonly used in voxel editors, game engines, and scientific visualization tools. The method described here is based on implementations in open-source projects like VXCT [S2, unofficial] and VRDF SDK [S1, unofficial].

## Sources
[S1] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S2] Helliaca/VXCT, https://github.com/Helliaca/VXCT, fetched 2026-10-26 (unofficial)
[S3] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S4] hsafari68/Voxelerate, https://github.com/hsafari68/Voxelerate, fetched 2026-10-26 (unofficial)
[S5] feat(sdf-wizard): [W] 20-Second SDF X-Ray & Measurement Wizard (uDebugVizMode 1..5 + live caliper), https://github.com/bourkefloyd/gemini-4-argon-sfhackathon-2026/pull/23, fetched 2026-10-26 (unofficial)
