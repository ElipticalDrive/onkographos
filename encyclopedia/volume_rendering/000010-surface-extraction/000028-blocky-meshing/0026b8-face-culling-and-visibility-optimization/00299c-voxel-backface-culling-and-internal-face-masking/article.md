# Voxel Backface Culling and Internal Face Masking

Also called: Internal Face Masking, Voxel Backface Culling

## Overview
This method optimizes voxel mesh generation by rejecting faces that are geometrically occluded by neighboring voxels (internal faces) and faces that are oriented away from the camera (backfaces). It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md) by adding view-dependent orientation checks and bit-packed visibility masks to reduce geometry sent to the rasterizer. Unlike simple neighbor checks, it uses projection-space transformations to determine face visibility relative to the view direction.

## How it works
### Inputs and Outputs
*   **Input**: A set of occupied voxel positions, typically organized into chunks (e.g., $32^3$ voxels per chunk) [S1, unofficial].
*   **Output**: A list of visible vertices/faces for rendering.

### Data Structures
The method relies on encoding visibility information into a compact integer format to facilitate fast checks in the geometry shader or vertex processing stage [S1, unofficial].

*   **Voxel Code (32-bit integer)**: Encodes position and face visibility masks.
    *   Bits 0-5: Mask for each of the 6 sides (1 bit per side).
    *   Bits 6-30: Position coordinates (3 * 8 bits, allowing $256^3$ addressable positions).
    *   Bits 31-32: Unused.
*   **Chunk**: A $32^3$ block of voxels used for efficient memory management and culling [S1, unofficial].

### Formulas
1.  **View Direction in Projection Space**:
    To determine if a face is front-facing or back-facing, the view direction vector must be transformed from projection space to view space. This is done by multiplying the projection space direction vector $v_{proj}$ by the inverse diagonal of the projection matrix $M_{proj}$ [S1, unofficial]:
    $$ v_{view} = v_{proj} \cdot \left( \frac{1}{M_{proj}[0][0]}, \frac{1}{M_{proj}[1][1]}, \frac{1}{M_{proj}[2][2]} \right) $$

2.  **Front/Back Face Determination**:
    For a voxel with corners $c_{vCorner000}$ and $c_{vCorner110}$ (representing opposite corners along the depth axis in the local voxel space), the dot product with the transformed view direction determines orientation [S1, unofficial]:
    $$ \text{IsFront} = \text{dot}\left( (c_{vCorner000}.xyz + c_{vCorner110}.xyz) \cdot c_{vInverseProjection}.xyz, v_{view} \right) > 0 $$
    *   If true, process the front face.
    *   If false, process the back face.

3.  **Internal Face Masking**:
    Faces shared between two neighboring voxels are never visible. Specific bits in the voxel code mask these internal faces [S1, unofficial]:
    *   Front face bit: `0x10` (bit 4).
    *   Back face bit: `0x20` (bit 5).
    *   Condition: If the respective bit is set, the face is internal and must be rejected.

### Pseudocode
The following pseudocode implements the logic described in the source, adapted for clarity. It assumes a geometry shader context where vertices are emitted.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_voxel_face(voxel_code, voxel_position, world_view_matrix, projection_matrix, view_direction):
    """
    Determines if a voxel face should be rendered based on backface culling and internal masking.
    
    Args:
        voxel_code: 32-bit integer containing position and face masks.
        voxel_position: (x, y, z) coordinates of the voxel.
        world_view_matrix: 4x4 matrix for transforming normals.
        projection_matrix: 4x4 projection matrix.
        view_direction: View direction vector in projection space.
        
    Returns:
        List of vertices to emit, or empty list if culled.
    """
    # Extract inverse projection diagonal
    inv_proj_diag = (
        1.0 / projection_matrix[0][0],
        1.0 / projection_matrix[1][1],
        1.0 / projection_matrix[2][2]
    )
    
    # Transform view direction to view space
    v_view = (
        view_direction.x * inv_proj_diag[0],
        view_direction.y * inv_proj_diag[1],
        view_direction.z * inv_proj_diag[2]
    )
    
    # Define voxel corners relative to position (assuming unit size)
    # c_vCorner000 is (x, y, z), c_vCorner110 is (x+1, y+1, z)
    # Note: Source uses specific corner indices for front/back determination
    c_vCorner000 = (voxel_position.x, voxel_position.y, voxel_position.z)
    c_vCorner110 = (voxel_position.x + 1, voxel_position.y + 1, voxel_position.z)
    
    # Calculate sum of corners for orientation test
    corner_sum = (
        c_vCorner000[0] + c_vCorner110[0],
        c_vCorner000[1] + c_vCorner110[1],
        c_vCorner000[2] + c_vCorner110[2]
    )
    
    # Dot product with inverse projection and view direction
    dot_product = (
        corner_sum[0] * inv_proj_diag[0] * v_view[0] +
        corner_sum[1] * inv_proj_diag[1] * v_view[1] +
        corner_sum[2] * inv_proj_diag[2] * v_view[2]
    )
    
    visible_vertices = []
    
    if dot_product > 0:
        # Front face logic
        # Check if front face bit (0x10) is set -> Internal face -> Cull
        if (voxel_code & 0x10) == 0:
            # Emit front face vertices: c_vCorner000, c_vCorner100, c_vCorner010, c_vCorner110
            # Normal is (0, 0, -1) transformed by world-view matrix
            normal = transform_normal((0, 0, -1), world_view_matrix)
            # Compute lighting and emit vertices
            # gl_Position = corner + vPos (where vPos is view-space offset)
            visible_vertices.extend([
                c_vCorner000,
                (voxel_position.x + 1, voxel_position.y, voxel_position.z), # c_vCorner100
                (voxel_position.x, voxel_position.y + 1, voxel_position.z), # c_vCorner010
                c_vCorner110
            ])
    else:
        # Back face logic
        # Check if back face bit (0x20) is set -> Internal face -> Cull
        if (voxel_code & 0x20) == 0:
            # Emit back face vertices: c_vCorner011, c_vCorner111, c_vCorner001, c_vCorner101
            # Normal is (0, 0, 1) transformed by world-view matrix
            normal = transform_normal((0, 0, 1), world_view_matrix)
            # Compute lighting and emit vertices
            visible_vertices.extend([
                (voxel_position.x, voxel_position.y + 1, voxel_position.z + 1), # c_vCorner011
                (voxel_position.x + 1, voxel_position.y + 1, voxel_position.z + 1), # c_vCorner111
                (voxel_position.x, voxel_position.y, voxel_position.z + 1), # c_vCorner001
                (voxel_position.x + 1, voxel_position.y, voxel_position.z + 1) # c_vCorner101
            ])
            
    return visible_vertices

def transform_normal(normal, matrix):
    """Helper to transform a normal vector by a 4x4 matrix."""
    # Simplified: assume matrix is 3x3 for normals or use inverse-transpose if non-uniform scaling
    # For standard view matrices, direct multiplication is often sufficient for direction vectors
    return (
        matrix[0][0]*normal[0] + matrix[1][0]*normal[1] + matrix[2][0]*normal[2],
        matrix[0][1]*normal[0] + matrix[1][1]*normal[1] + matrix[2][1]*normal[2],
        matrix[0][2]*normal[0] + matrix[1][2]*normal[1] + matrix[2][2]*normal[2]
    )
```

### Edge Cases and Pitfalls
*   **Internal Faces**: Shared faces between two neighboring voxels are never visible. The masking bits (`0x10` and `0x20`) are critical for rejecting these invisible internal faces [S1, unofficial].
*   **Projection Space Transformation**: The view direction must be transformed using the inverse diagonal of the projection matrix. Using the raw view direction in projection space without this transformation will result in incorrect backface culling [S1, unofficial].
*   **Chunk Boundaries**: The method uses a $32^3$ chunk size. Care must be taken when handling voxels at chunk boundaries to ensure neighbor checks (for internal masking) are performed correctly across chunk borders [S1, unofficial].
*   **Performance**: The original implementation achieved 32,368 voxels per millisecond on a Geforce 640M LE [S1, unofficial]. Performance is highly dependent on the efficiency of the geometry shader and the compactness of the voxel code encoding.

## Operating characteristics
*   **Time Complexity**: $O(V)$ where $V$ is the number of voxels, as each voxel is processed once. The constant factor is low due to bit-packed checks.
*   **Memory Complexity**: $O(V)$ for storing voxel positions and codes. The 32-bit encoding is efficient.
*   **Performance**: The source reports 32,368 voxels per millisecond on a Geforce 640M LE [S1, unofficial].
*   **Strengths**: Reduces geometry significantly by culling both internal and back-facing faces. The bit-packed encoding allows for fast checks in the shader.
*   **Limitations**: Requires a specific projection matrix structure for the inverse diagonal trick. Chunked approach is necessary for large worlds due to the 8-bit position encoding limit ($256^3$).

## Background
This method was described by Johannes Jendersie in a graphic development blog post [S1, unofficial]. It was part of a voxel rendering engine ("Monolith") that used geometry shaders for mesh generation. The technique was designed to optimize rendering performance by reducing the number of primitives sent to the rasterizer.

## Sources
[S1] Graphic | Developerblog – Johannes Jendersie, https://jojendersie.de/category/graphic/, fetched 2026-10-24 (unofficial)
