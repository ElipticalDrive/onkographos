# Instance Data Layout

Also called: ShapeInstance Struct Layout

## Overview
A tightly packed data structure used to pass instance data (position, texture, tint, shape ID, orientation) to the GPU for instanced rendering of voxel shapes [S1, unofficial]. It builds on [Global Uniform Data Layout](2955-global-uniform-data-layout.md) by defining the specific memory layout for individual shape instances rather than global rendering parameters. It differs from [Camera Uniform Layout](3619-camera-uniform-layout.md) and [Appearance Frame Uniform Layout](12147-appearance-frame-uniform-layout.md) by focusing on per-instance geometric and material properties rather than view or frame-level state.

## How it works
**Inputs and Outputs:**
*   **Input:** Instance parameters including 3D position, texture index, shape identifier, rotation state, and color tint.
*   **Output:** A 32-byte binary blob suitable for GPU buffer binding (e.g., `GL_ARRAY_BUFFER` or `GL_SHADER_STORAGE_BUFFER`).

**Data Structures:**
The core structure is `ShapeInstance`, defined with `LayoutKind.Explicit` and `Pack = 1` to ensure no padding bytes are inserted by the compiler [S1, unofficial].

| Field | Type | Size | Offset | Description |
| :--- | :--- | :--- | :--- | :--- |
| `position` | `Vector3` (float[3]) | 12 bytes | 0 | World position of the instance. |
| `texture` | `int` | 4 bytes | 12 | Index into the texture array. |
| `shape` | `ushort` | 2 bytes | 16 | Unique shape identifier. |
| `rotation` | `ushort` | 2 bytes | 18 | Encoded rotation state. |
| `tint` | `Vector3` (float[3]) | 12 bytes | 20 | Color tint applied to the instance. |

**Total Size:** 32 bytes.

**Rotation Encoding:**
The `rotation` field uses a compact encoding scheme [S1, unofficial]:
$$ \text{rotation} = (\text{up} \times 4) + \text{forward} $$
Where:
*   `up` is an enum value representing the primary axis direction (mapped to integer indices).
*   `forward` is an integer in the range $[0, 3]$ representing clockwise rotation perpendicular to the `up` axis.

**Shader Integration:**
In shader code, the `shape` (2 bytes) and `rotation` (2 bytes) fields are combined into a single 32-bit unsigned integer (`uint`) to optimize sampling or indexing [S1, unofficial]. This is typically done by shifting the shape ID and OR-ing with the rotation value:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for shader-side reconstruction
shape_rotation_uint = (shape_id << 16) | rotation_value
```

**Edge Cases and Pitfalls:**
*   **Shape ID Truncation:** The `shape` field is truncated to `ushort` (2 bytes), limiting the number of unique shapes to 65,535 [S1, unofficial]. Although the constructor may accept an `int`, values exceeding 65535 will wrap or truncate, causing incorrect shape assignment.
*   **Alignment:** The layout relies on explicit packing. If the compiler or GPU driver inserts padding (e.g., default pack alignment), the offsets will be incorrect. Explicit layout directives must be enforced on the CPU side.

## Subtopics

### ShapeInfo Data Layout
A more detailed structure containing full geometric parameters for various primitive types [S2, unofficial].
*   **Layout:** `struct ShapeInfo` contains: `position` (Vector3), `shape` (int), `color` (Vector3), `sphereRadius` (float), `boxDimensions` (Vector3), `roundBoxDimensions` (Vector3), `roundBoxFactor` (float), `torusOuterRadius` (float), `torusInnerRadius` (float), `coneHeight` (float), `coneRatio` (Vector2).
*   **Size:** Total size is 80 bytes. Calculated as 19 floats ($19 \times 4 = 76$ bytes) + 1 int (4 bytes).
*   **Packing:** Fields are packed in declaration order. Vector3 uses 3 floats, Vector2 uses 2 floats.
*   **Usage:** Used in volumetric ray casting to define the geometry of individual shapes during ray intersection tests [S2, unofficial].

## Operating characteristics
*   **Memory:** 32 bytes per instance. Very compact, allowing high instance counts within GPU memory limits.
*   **Performance:** The compact size reduces memory bandwidth usage during instanced draw calls. The combined `shapeRotation` uint allows for efficient bit-field extraction in shaders.
*   **Limitations:** The 16-bit shape ID limit (65,535 shapes) may be insufficient for highly complex scenes requiring many distinct voxel shapes. The rotation encoding is limited to 12 discrete orientations (3 up directions $\times$ 4 forward rotations), which may not support arbitrary continuous rotation without additional data.

## Background
This layout originates from the `Cubic-Grid_Voxel_Renderer` project [S1, unofficial]. It was designed to efficiently pass instance data to the GPU for rendering voxel-based shapes using instanced rendering techniques. The explicit packing ensures compatibility with OpenGL buffer objects.

## Sources
[S1] rarefushion/Cubic-Grid_Voxel_Renderer, https://github.com/rarefushion/Cubic-Grid_Voxel_Renderer, fetched 2026-10-27 (unofficial)
[S2] Kobakat/VolumetricRayCasting, https://github.com/Kobakat/VolumetricRayCasting, fetched 2026-10-26 (unofficial)
