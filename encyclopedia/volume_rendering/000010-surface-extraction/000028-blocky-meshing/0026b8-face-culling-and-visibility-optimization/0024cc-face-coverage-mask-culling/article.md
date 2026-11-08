# Face Coverage Mask Culling

Also called: Bitmask Culling, Partial Shape Culling

## Overview
Face Coverage Mask Culling is a method for removing hidden faces of partial voxel shapes (such as stairs, slabs, or custom geometries) by representing face coverage as a bit mask [S1, unofficial]. It solves the problem where a boolean "solid/empty" flag is insufficient because partial shapes have mixed visible and hidden areas on a single face [S1, unofficial]. Unlike sibling methods that rely on coarse block visibility or vertex packing, this method operates at the sub-face granularity, allowing precise culling of geometry that is occluded by neighbors while preserving exposed portions of the same face [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D grid of block states, where each state contains a shape identifier, an orientation (rotation), and a precomputed occlusion map [S1, unofficial]. The occlusion map maps each of the 6 face directions and 4 rotations to a `FaceMask` [S1, unofficial].
*   **Output:** A set of visible faces for the mesh, where any face fully occluded by a neighbor is removed [S1, unofficial].

**Data Structures:**
The core data structure is `FaceMask`, a 16-bit unsigned integer representing a 4×4 grid of cells on a face boundary [S1, unofficial].
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class FaceMask:
    def __init__(self, value: int):
        self.value = value
```
Constants define the extremes of coverage:
*   `FULL`: `FaceMask(0xffff)` (all 16 bits set), representing a fully opaque face [S1, unofficial].
*   `EMPTY`: `FaceMask(0x0000)` (all 16 bits unset), representing a fully empty face [S1, unofficial].

The block state stores occlusion data as a lookup table: `shape.occlusion[rotation][face]` returns the `FaceMask` for a specific rotation and face direction [S1, unofficial].

**Algorithm Steps:**
1.  **Retrieve Masks:** For a given voxel face, retrieve the current block's coverage mask and the neighbor's opposite coverage mask [S1, unofficial].
    *   Let $M_{curr}$ be `shape.occlusion[rotation][face]` [S1, unofficial].
    *   Let $M_{nbr}$ be `neighbor_shape.occlusion[neighbor_rotation][opposite(face)]` [S1, unofficial].
2.  **Calculate Visibility:** Determine if the current face is fully occluded by the neighbor [S1, unofficial]. A face is fully occluded if the neighbor covers every bit that is set in the current mask [S1, unofficial].
    *   Formula: `visible = M_curr & ~M_nbr`
    *   Condition: If `visible == 0` (or equivalently `M_curr & ~M_nbr == 0`), the face is fully occluded [S1, unofficial].
3.  **Cull or Keep:**
    *   If fully occluded, discard the face geometry entirely [S1, unofficial].
    *   If partially visible (`visible != 0`), the implementer must choose between two strategies [S1, unofficial]:
        1.  **Keep Original Geometry:** Emit the full face geometry, accepting overdraw in the transparent/depth buffer [S1, unofficial].
        2.  **Subdivide Geometry:** Generate new geometry that only covers the bits set in `visible` [S1, unofficial]. This reduces geometry count but increases mesher complexity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cull_face(shape, neighbor, rotation, face_direction):
    """
    Determines if a face is fully occluded by a neighbor.
    
    Args:
        shape: Object with .occlusion table [rotation][face] -> FaceMask
        neighbor: Object with .occlusion table and .rotation attribute, or None
        rotation: int, current block rotation
        face_direction: int, direction of the face (0-5)
        
    Returns:
        bool: True if the face is fully occluded and should be removed.
    """
    # 1. Retrieve current mask
    current_mask = shape.occlusion[rotation][face_direction]
    
    # 2. Retrieve neighbor mask
    if neighbor is None:
        neighbor_mask = FaceMask(0x0000)
    else:
        # Standard opposite mapping for 6 face directions [G]
        opposite_faces = {0: 1, 1: 0, 2: 3, 3: 2, 4: 5, 5: 4}
        opposite_face = opposite_faces[face_direction]
        neighbor_mask = neighbor.occlusion[neighbor.rotation][opposite_face]
        
    # 3. Check for full occlusion
    visible_bits = current_mask.value & ~neighbor_mask.value
    
    return visible_bits == 0
```

**Parameters:**
*   **Grid Resolution:** The `FaceMask` assumes a 4×4 subdivision [S1, unofficial]. This resolution is sufficient for standard partial shapes but may need adjustment for higher-fidelity custom shapes [S1, unofficial].
*   **OcclusionMode:** For transparent materials, the culling logic is extended by an `OcclusionMode` enum (opaque, cutout, translucent, never-occluding) combined with the coverage mask [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Partial Visibility:** If a face is only partially occluded, simply removing the face removes visible geometry [S1, unofficial]. The implementer must handle partial visibility by either keeping the full face (overdraw) or subdividing the mesh [S1, unofficial].
*   **Missing Neighbors:** If a neighbor chunk is not loaded, the engine must define a policy: treat as air and remesh later, delay meshing, or use an explicit "unknown" state [S1, unofficial]. The correct choice depends on the world-streaming model [S1, unofficial].
*   **Internal Geometry:** Culling logic should only apply to geometry lying on the voxel cell boundary planes [S1, unofficial]. Internal triangles within a complex block should not be tested against neighbors [S1, unofficial].

## Subtopics

### [Face-Culling Geometry Shader](3928-face-culling-geometry-shader.md)
A geometry shader technique that generates cube faces from point primitives based on a bitmask, culling hidden faces to reduce overdraw and vertex processing. It solves the problem of rendering voxel surfaces by emitting only the visible faces of a voxel, leveraging the GPU's geometry shader stage to expand point data into full cube geometry on-demand. It builds on Face Coverage Mask Culling by using the bitmask to determine face visibility at the primitive generation stage.

### [Per-face Column-based Face Culling with Opacity Masks](4821-per-face-column-based-face-culling-with-opacity-masks.md)
A face culling technique that uses per-face column arrays and opaque column arrays to determine visibility based on solid and transparent block transitions, allowing full per-face control over opacity. It solves the problem where a boolean solid flag is insufficient because partial shapes have mixed visible/hidden areas on a single face, extending Face Coverage Mask Culling by distinguishing between opaque and transparent surface contributions.

### [Boundary Geometry Classification](5435-boundary-geometry-classification.md)
Boundary Geometry Classification is a filtering step in blocky meshing that ensures neighbor-based face culling is applied exclusively to geometry lying on voxel cell boundary planes, while leaving internal geometry of complex blocks untouched. It solves the problem where culling logic intended for inter-block occlusion would incorrectly suppress valid internal triangles within a single complex shape. It differs from Face Coverage Mask Culling by operating on the geometric position of vertices rather than precomputed coverage bitmasks, and from Face-Culling Geometry Shader by being a CPU-side classification logic rather than a GPU shader stage.

## Operating characteristics
*   **Time Complexity:** The culling decision per face is $O(1)$, involving only bitwise operations and table lookups [S1, unofficial].
*   **Memory Complexity:** Requires storing a `FaceMask` (2 bytes) per face per rotation per shape [S1, unofficial]. Precomputing masks for all rotations and shapes increases memory usage but speeds up the meshing loop [S1, unofficial].
*   **Performance:** The method is highly efficient for the common path because the representation makes the common path cheap [S1, unofficial]. Good properties include precomputed shape masks, precomputed rotation variants, small integer masks, contiguous block-state storage, cheap neighbor access, and minimal allocation inside the inner loop [S1, unofficial]. It scales well with large volumes because it reduces primitive count significantly for partial shapes [S1, unofficial].
*   **Strengths:** Precise culling for non-cube shapes; reduces overdraw and geometry count; simple bitwise logic [S1, unofficial].
*   **Limitations:** Requires precomputed masks for all shapes and rotations; partial visibility handling adds complexity; does not handle transparency semantics natively without extension [S1, unofficial].

## Background
Face Coverage Mask Culling originated from the need to efficiently render partial voxel shapes (like stairs and slabs) in voxel engines [S1, unofficial]. Traditional boolean culling fails for these shapes because a face may be partially occupied [S1, unofficial]. The method was described in a technical blog post addressing the limitations of simple solid/empty flags for voxel meshing optimization [S1, unofficial].

## Sources
[S1] Face Culling for Voxel Cubes and Partial Shapes, https://ogyrecdev.github.io/posts/face-culling/, fetched 2026-10-28 (unofficial)
