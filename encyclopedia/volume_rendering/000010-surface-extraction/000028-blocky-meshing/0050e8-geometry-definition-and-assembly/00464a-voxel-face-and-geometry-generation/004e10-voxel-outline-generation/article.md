# Voxel Outline Generation

Also called: Line Outline Generation, Voxel Joint Outline

## Overview
Voxel Outline Generation renders the edges of a voxel grid as line primitives to emphasize the blocky structure of the model. It solves the visual need for clear geometric definition in voxel art by drawing lines around the perimeter and holes of rectangular faces. It differs from [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) by emitting line segments instead of quads, and from [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by focusing on aesthetic edge highlighting rather than precise surface topology.

## How it works
**Inputs:** 2D color planes representing voxel layers (derived from the parent's neighbor check and face culling).
**Outputs:** A list of line segments (pairs of 3D vertices).

**Data Structures:**
*   **Color Planes:** 2D arrays where each cell represents a voxel face color or empty state.
*   **Line Buffer:** A dynamic list of `vec3` pairs.

**Steps:**
1.  **Iterate Planes:** Process the 2D color planes used for face merging [S1, unofficial].
2.  **Generate Lines:** For each rectangular face (including holes within larger faces), generate line segments around its perimeter [S1, unofficial].
    *   The algorithm examines the 12 edges of each voxel to determine if they should be drawn based on the presence and color of neighboring voxels [S1, unofficial].
    *   Alternatively, lines are generated directly around the perimeter and holes of the rectangular faces in the 2D planes [S1, unofficial].
3.  **Depth Bias:** When rendering the lines, apply a slight depth buffer bias to avoid z-fighting and ensure lines are visible [S1, unofficial].

**Pitfalls:**
*   **Duplicate Segments:** The process generates duplicate line segments (exactly 2x) because adjacent faces share edges. These are typically not filtered out as the visual impact is negligible [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_outlines(color_planes):
    lines = []
    # Iterate through each 2D color plane representing a slice of voxels
    for plane in color_planes:
        # Identify rectangular faces and their holes
        # This step relies on the parent's face culling logic
        faces = extract_rectangular_faces(plane)
        
        for face in faces:
            # Generate line segments around the perimeter and holes of the face
            perimeter_lines = generate_perimeter_lines(face)
            lines.extend(perimeter_lines)
            
    return lines

def apply_depth_bias(line, bias_factor=0.001):
    # Adjust vertex positions slightly along the view direction to prevent z-fighting
    # Implementation depends on the specific renderer's depth buffer handling
    pass
```

## Subtopics

### Voxel Grid Overlay
Draws a grid pattern on voxel faces using UV coordinates and a thickness uniform, without requiring a grid texture [S2, unofficial].
*   **Definition:** A shader-based technique that overlays a grid on existing voxel faces [S2, unofficial].
*   **Steps:**
    1.  If `uGridThickness > 0.0`:
    2.  Compute `uv = fract(vModelUv)`.
    3.  Compute `fromCenter = abs(uv - 0.5)`.
    4.  Compute `isEdge = step(0.5 - uGridThickness, fromCenter)`.
    5.  Mix `diffuseColor` with `materialColor + uGridColor` weighted by `max(isEdge.x, isEdge.y)` [S2, unofficial].
*   **Parameters:**
    *   `uGridThickness`: float uniform controlling grid line width [S2, unofficial].
    *   `uGridColor`: vec3 uniform specifying the grid line color [S2, unofficial].

## Operating characteristics
*   **Complexity:** Linear with respect to the number of exposed faces.
*   **Performance:** Generates significantly less geometry than face meshing (lines vs. quads). In a test with 226,036 voxels, it produced 4,682 lines compared to 29,666 triangles [S1, unofficial].
*   **Strengths:** Simple to implement; provides clear visual separation between voxels; low memory footprint.
*   **Limitations:** Does not filter duplicate edges, leading to redundant draw calls or vertex data; requires depth bias to avoid rendering artifacts.

## Background
Originated from software rendering experiments for voxel models, specifically in the context of learning graphics programming [S1, unofficial]. It is used in voxel art engines and educational graphics projects to provide a clean, blocky aesthetic.

## Sources
[S1] Voxel Rendering Techniques - Medium, https://medium.com/@fogleman/voxel-rendering-techniques-fa8d869457ca, fetched 2026-10-27 (unofficial)
[S2] aresrpg/aresrpg-engine, https://github.com/aresrpg/aresrpg-engine, fetched 2026-10-27 (unofficial)
