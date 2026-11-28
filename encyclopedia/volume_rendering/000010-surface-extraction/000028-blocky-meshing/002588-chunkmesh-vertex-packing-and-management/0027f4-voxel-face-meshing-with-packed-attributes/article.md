# Voxel Face Meshing with Packed Attributes

Also called: Packed Voxel Face Meshing, Voxel Meshing with Packed Vertex Attributes

## Overview
A mesh generation technique for voxel terrain where vertex data (position, type, direction) is packed into a single float attribute to reduce buffer overhead, unpacked in the vertex shader, and used to generate quad faces for visible voxel surfaces [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a chunk of voxel data, a packed buffer containing one float per vertex, and uniform variables for chunk and world transformation as inputs [S1, unofficial]. It outputs a `BufferGeometry` containing a `position` attribute (vec3), a `packedData` attribute (float), a `vertexID` attribute (float), and an index buffer storing triangle indices for the generated quads [S1, unofficial].

**Data Structures:**
The `packed_buffer` stores vertex attributes in a single float per vertex [S1, unofficial]. The bit layout packs the data as follows: bits 20–23 (4 bits) store the X coordinate within the voxel face (0–15), bits 16–19 (4 bits) store the Y coordinate (0–15), bits 12–15 (4 bits) store the Z coordinate (0–15), bits 4–11 (8 bits) store the voxel type (0–255), and bits 0–2 (3 bits) store the face direction (0–7) [S1, unofficial].

**Parameters:**
The method relies on four uniform parameters: `u_voxel_size` (float, size of a single voxel in world units), `u_chunk_size` (float, number of voxels per chunk dimension), `u_chunk_indices` (vec3, integer coordinates of the chunk in the world grid), and `u_time` (float, elapsed time for water animation) [S1, unofficial].

**Steps:**
1. **Unpack Data:** In the vertex shader, extract the components from the packed float using bitwise division and modulo operations [S1, unofficial].
   $$x = \lfloor \text{mod}(packed / 2^{20}, 16) \rfloor$$
   $$y = \lfloor \text{mod}(packed / 2^{16}, 16) \rfloor$$
   $$z = \lfloor \text{mod}(packed / 2^{12}, 16) \rfloor$$
   $$type = \lfloor \text{mod}(packed / 2^{4}, 256) \rfloor$$
   $$direction = \text{mod}(packed, 8)$$
2. **Calculate Local Offset:** Determine the vertex offset relative to the voxel center based on the face direction and vertex ID [S1, unofficial]. The half-size is calculated as $u\_voxel\_size \times 0.5$ [S1, unofficial]. The supported directions are 0=TOP, 1=BOTTOM, 2=RIGHT, 3=LEFT, 4=FRONT, and 5=BACK [S1, unofficial].
3. **Compute World Position:** Calculate the final world position by combining the chunk offset, the voxel local offset, and the face vertex offset [S1, unofficial].
   $$chunk\_offset = u\_chunk\_indices \times (u\_chunk\_size \times u\_voxel\_size)$$
   $$local\_offset = voxelPosition \times u\_voxel\_size$$
   $$vertexOffset = \text{getFaceVertexPosition}(direction, vertexID)$$
   $$final\_position = chunk\_offset + local\_offset + vertexOffset$$
4. **Apply Effects:** If the voxel type is 5.0 (water) and the vertex position Y coordinate is greater than 0.0, apply a ripple displacement to the Y axis [S1, unofficial].
   $$position.y += \sin(worldPos.x \times 2.0 + u\_time \times 1.5) \times 0.05 + \cos(worldPos.z \times 2.0 + u\_time \times 2.0) \times 0.06$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def unpack_data(packed: float) -> tuple:
    """Unpacks the packed float into position, type, and direction [S1, unofficial]."""
    x = math.floor(math.fmod(packed / 2.0**20.0, 16.0))
    y = math.floor(math.fmod(packed / 2.0**16.0, 16.0))
    z = math.floor(math.fmod(packed / 2.0**12.0, 16.0))
    type_val = math.floor(math.fmod(packed / 2.0**4.0, 256.0))
    direction = math.fmod(packed, 8.0)
    return (x, y, z), type_val, direction

def get_face_vertex_position(direction: int, vertex_id: int, voxel_size: float) -> tuple:
    """Calculates the offset of a specific vertex on a face relative to the voxel center [G]."""
    half = voxel_size * 0.5
    # Standard counter-clockwise quad vertex ordering for each face relative to voxel center [G]
    offsets = {
        0: [(-half, half, -half), (half, half, -half), (half, half, half), (-half, half, half)], # TOP
        1: [(-half, -half, half), (half, -half, half), (half, -half, -half), (-half, -half, -half)], # BOTTOM
        2: [(half, -half, -half), (half, half, -half), (half, half, half), (half, -half, half)], # RIGHT
        3: [(-half, -half, half), (-half, half, half), (-half, half, -half), (-half, -half, -half)], # LEFT
        4: [(-half, -half, half), (half, -half, half), (half, half, half), (-half, half, half)], # FRONT
        5: [(half, -half, -half), (-half, -half, -half), (-half, half, -half), (half, half, -half)]  # BACK
    }
    return offsets[direction][vertex_id]

def compute_world_position(chunk_indices: tuple, chunk_size: float, voxel_size: float,
                           voxel_pos: tuple, direction: int, vertex_id: int,
                           voxel_type: float, time: float) -> tuple:
    """Computes the final world position for a vertex [S1, unofficial]."""
    chunk_offset = (chunk_indices[0] * chunk_size * voxel_size,
                    chunk_indices[1] * chunk_size * voxel_size,
                    chunk_indices[2] * chunk_size * voxel_size)
    local_offset = (voxel_pos[0] * voxel_size, voxel_pos[1] * voxel_size, voxel_pos[2] * voxel_size)
    vertex_offset = get_face_vertex_position(direction, vertex_id, voxel_size)

    final_pos = (chunk_offset[0] + local_offset[0] + vertex_offset[0],
                 chunk_offset[1] + local_offset[1] + vertex_offset[1],
                 chunk_offset[2] + local_offset[2] + vertex_offset[2])

    # Water ripple effect applies only to top faces to avoid distorting submerged or side faces [S1, unofficial]
    if voxel_type == 5.0 and final_pos[1] > 0.0:
        ripple = math.sin(final_pos[0] * 2.0 + time * 1.5) * 0.05 + \
                 math.cos(final_pos[2] * 2.0 + time * 2.0) * 0.06
        final_pos = (final_pos[0], final_pos[1] + ripple, final_pos[2])

    return final_pos
```

**Edge Cases and Pitfalls:**
The water ripple effect only applies to top faces (`position.y > 0.0`) to avoid distorting submerged or side faces [S1, unofficial].

## Operating characteristics
*   **Memory Complexity:** Reduces buffer overhead by packing multiple attributes into a single float per vertex [S1, unofficial].
*   **Limitations:** Limited precision for coordinates (4 bits per axis), limited voxel types (8 bits), and limited directions (3 bits) [S1, unofficial].
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **When to use:** Not given in the sources [S1, unofficial].
*   **When not to use:** Not given in the sources [S1, unofficial].

## Background
Not given in the sources [S1, unofficial].

## Sources
[S1] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
