# Voxel Face Normal and Offset Mapping

Also called: Voxel Face Enum Mapping

## Overview
This method provides the geometric and topological mapping between a discrete voxel face identifier and its continuous 3D properties. It solves the problem of translating a logical face index (e.g., "top" or "left") into a unit normal vector for shading/lighting calculations and an integer offset vector for neighbor voxel access during surface extraction. It differs from [Face Direction and Quad Geometry](4881-face-direction-and-quad-geometry.md) by focusing on the data mapping logic rather than the mesh assembly primitives.

## How it works
**Inputs:** A face identifier, typically an enumeration (`EVoxelFace` or integer index).
**Outputs:** A 3D unit normal vector $\mathbf{n}$ and a 3D integer offset vector $\mathbf{o}$.

**Data Structures:**
The face identifier is an integer. Two common layouts exist:
1.  **Standard 6-face:** Indices 0–5 representing the six cube faces.
2.  **Extended (Sprite) indices:** Indices 6–7 reserved for sprite-specific faces [S2, unofficial].

**Mapping Logic:**
The mapping is a direct lookup based on the face direction. The normal $\mathbf{n}$ and offset $\mathbf{o}$ are identical for the standard 6-face configuration.

1.  **Normal Vector Mapping:**
    *   Top: $\mathbf{n} = (0, 0, 1)$
    *   Bottom: $\mathbf{n} = (0, 0, -1)$
    *   North: $\mathbf{n} = (0, 1, 0)$
    *   South: $\mathbf{n} = (0, -1, 0)$
    *   East: $\mathbf{n} = (1, 0, 0)$
    *   West: $\mathbf{n} = (-1, 0, 0)$
    *   Default/Invalid: $\mathbf{n} = (0, 0, 0)$ [S1, unofficial]

2.  **Offset Vector Mapping:**
    The offset $\mathbf{o}$ is used to access the neighbor voxel in the direction of the face normal. It is identical to the normal vector but interpreted as integer coordinates [S1, unofficial].
    *   Top: $\mathbf{o} = (0, 0, 1)$
    *   Bottom: $\mathbf{o} = (0, 0, -1)$
    *   North: $\mathbf{o} = (0, 1, 0)$
    *   South: $\mathbf{o} = (0, -1, 0)$
    *   East: $\mathbf{o} = (1, 0, 0)$
    *   West: $\mathbf{o} = (-1, 0, 0)$
    *   Default/Invalid: $\mathbf{o} = (0, 0, 0)$ [S1, unofficial]

3.  **Opposite Face Calculation:**
    To find the opposite face direction for a given face $F$:
    *   Top $\leftrightarrow$ Bottom
    *   North $\leftrightarrow$ South
    *   East $\leftrightarrow$ West
    *   Default $\rightarrow$ Default [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_face_normal_and_offset(face_id: int):
    """
    Returns (normal: Vector3, offset: Vector3) for a given face ID.
    face_id: Integer representing the face direction.
    """
    # Standard 6-face mapping
    if face_id == 0: # Top
        return (Vector3(0, 0, 1), Vector3(0, 0, 1))
    elif face_id == 1: # Bottom
        return (Vector3(0, 0, -1), Vector3(0, 0, -1))
    elif face_id == 2: # North
        return (Vector3(0, 1, 0), Vector3(0, 1, 0))
    elif face_id == 3: # South
        return (Vector3(0, -1, 0), Vector3(0, -1, 0))
    elif face_id == 4: # East
        return (Vector3(1, 0, 0), Vector3(1, 0, 0))
    elif face_id == 5: # West
        return (Vector3(-1, 0, 0), Vector3(-1, 0, 0))
    else:
        # Default/Invalid case
        return (Vector3(0, 0, 0), Vector3(0, 0, 0))

def get_opposite_face(face_id: int):
    """
    Returns the index of the opposite face.
    """
    if face_id == 0: return 1
    elif face_id == 1: return 0
    elif face_id == 2: return 3
    elif face_id == 3: return 2
    elif face_id == 4: return 5
    elif face_id == 5: return 4
    else: return face_id
```

**Parameters:**
*   **Face Indexing:** In some implementations (e.g., [S2, unofficial]), faces are indexed 0–5: `V3_FACE_LEFT=0`, `V3_FACE_RIGHT=1`, `V3_FACE_TOP=2`, `V3_FACE_BOTTOM=3`, `V3_FACE_FRONT=4`, `V3_FACE_BACK=5`. Note that the mapping of indices to directions may vary between implementations (e.g., [S1, unofficial] uses Top=0, [S2, unofficial] uses Left=0). The implementer must verify the specific enum definition in their codebase.
*   **Sprite Indices:** Indices 6–7 (`V3_SPRITE_RIGHT`, `V3_SPRITE_LEFT`) are used for sprites and do not map to standard voxel normals [S2, unofficial].

**Edge Cases:**
*   **Invalid Face ID:** If the input is not 0–5, the method returns a zero vector. This should be handled by the caller to avoid invalid neighbor access or shading artifacts.
*   **Index Mismatch:** The mapping of integer indices to directions (e.g., is 0 Top or Left?) is not standardized across all voxel engines. The implementer must align the `face_id` with the specific enum definition used in the geometry generation step.

## Subtopics

### Face Identifier Enum
The `EVoxelFace` enum (or equivalent integer constants) defines the logical face directions. The specific assignment of integers to directions (e.g., whether 0 is Top or Left) varies by implementation. [S1, unofficial] uses Top=0, while [S2, unofficial] uses Left=0. This variation affects the lookup table in `get_face_normal_and_offset`. The enum must be consistent with the [Face Direction and Quad Geometry](4881-face-direction-and-quad-geometry.md) step to ensure normals align with generated quad vertices.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per face. The operation is a simple switch or array lookup.
*   **Memory Complexity:** $O(1)$. No additional memory is required beyond the input face ID.
*   **Strengths:** Extremely fast; no branching overhead if implemented as a lookup table.
*   **Limitations:** Requires the face ID to be valid. Incorrect enum mapping leads to incorrect normals and neighbor access.
*   **When to use:** During surface extraction, for every exposed face, this mapping is required to determine the face normal for lighting and the neighbor coordinate for boundary checks.

## Background
This method is a fundamental component of voxel-based terrain rendering, particularly in blocky meshing approaches where geometry is generated per-face. It originates from early voxel engines and is widely used in modern voxel libraries (e.g., [S1, unofficial], [S2, unofficial]) to decouple logical face identification from geometric representation.

## Sources
[S1] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S2] myusernameisbetter/42-Vox, https://github.com/myusernameisbetter/42-Vox, fetched 2026-10-27 (unofficial)
