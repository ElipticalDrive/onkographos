# Appearance Frame Uniform Layout

Also called: AppearanceFrame, Vulkan Appearance Frame Uniform Layout

## Overview
The Appearance Frame Uniform Layout is a specific `std140` uniform buffer structure used in Vulkan-based voxel rendering to pass camera, lighting, and brush state data to fragment shaders. It solves the problem of efficiently packing heterogeneous uniform data (matrices, vectors, and large arrays) into a single buffer for per-frame updates. It differs from sibling layouts like the [Camera Uniform Layout](3619-camera-uniform-layout.md) by including lighting and brush-specific fields, and from the [Instance Data Layout](3992-instance-data-layout.md) by focusing on global frame state rather than per-object transforms.

## How it works
**Inputs and Outputs:**
The method takes a sequence of uniform variables and packs them into a 2336-byte buffer. The output is a GLSL `uniform` block accessible by shaders.

**Data Structures:**
The buffer uses `std140` layout rules. Offsets are aligned to 16-byte boundaries. The total size is 2336 bytes (584 32-bit words).

**Fields and Layout:**
The fields are ordered as follows, with offsets calculated using 16-byte alignment padding:

1.  `camera_clip`: `mat4` (64 bytes)
2.  `light_clip`: `mat4` (64 bytes)
3.  `eye_near`: `vec4` (16 bytes)
4.  `camera_right`: `vec4` (16 bytes)
5.  `camera_up`: `vec4` (16 bytes)
6.  `camera_forward_focal`: `vec4` (16 bytes)
7.  `viewport`: `vec4` (16 bytes)
8.  `brush_cm`: `vec4` (16 bytes)
9.  `flags`: `uvec4` (16 bytes)
10. `palette`: `vec4[5]` (80 bytes)
11. `shadow_params`: `vec4` (16 bytes)
12. `selected`: `uvec4[125]` (2000 bytes)

**Offset Calculation Formula:**
For each field, the offset is calculated as:
$$ \text{offset} = \left\lfloor \frac{\text{offset} + 15}{16} \right\rfloor \times 16 $$
The size of a field is:
$$ \text{size} = \begin{cases} 64 & \text{if kind} == \text{'mat4'} \\ 16 & \text{otherwise} \end{cases} \times \text{count} $$
The final offset after the last field must equal 2336.

**Pseudocode:**
The following Python-like code demonstrates how the layout is generated, iterating through fields to compute offsets and sizes.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_layout():
    fields = [
        ("camera_clip", "mat4", 1),
        ("light_clip", "mat4", 1),
        ("eye_near", "vec4", 1),
        ("camera_right", "vec4", 1),
        ("camera_up", "vec4", 1),
        ("camera_forward_focal", "vec4", 1),
        ("viewport", "vec4", 1),
        ("brush_cm", "vec4", 1),
        ("flags", "uvec4", 1),
        ("palette", "vec4", 5),
        ("shadow_params", "vec4", 1),
        ("selected", "uvec4", 125),
    ]
    
    current_offset = 0
    layout_info = []
    
    for name, kind, count in fields:
        # Align to 16 bytes
        current_offset = (current_offset + 15) // 16 * 16
        
        # Calculate size
        if kind == 'mat4':
            size = 64 * count
        else:
            size = 16 * count
            
        layout_info.append({
            "name": name,
            "offset": current_offset,
            "size": size
        })
        
        current_offset += size
        
    assert current_offset == 2336, f"Total size mismatch: {current_offset}"
    return layout_info
```

**Parameters:**
*   Total uniform buffer size: 2336 bytes.
*   Total word count: 584 words (32-bit).

**Edge Cases and Pitfalls:**
*   **Precision:** The `std140` layout ensures consistent padding across drivers, but manual packing must respect the 16-byte alignment rule strictly.
*   **Boundaries:** The `selected` array is large (2000 bytes); ensure the buffer binding size is sufficient.
*   **Degenerate Input:** If `count` is 0 for `mat4`, the size is 0, but alignment still applies to the next field.

## Subtopics

### OperationInfo Data Layout
The `OperationInfo` data layout is a compact structure used for operation metadata. It contains `operation` (int), `childCount` (int), and `blendStrength` (float). The total size is 12 bytes (`sizeof(int) * 2 + sizeof(float) * 1`). It is used to store per-operation state, distinct from the global frame uniform data.

## Operating characteristics
**Time and Memory Complexity:**
Memory complexity is fixed at 2336 bytes per frame. Time complexity for updating the buffer is $O(1)$ relative to the number of pixels, but depends on the number of uniform updates per frame.

**Measured Performance:**
Performance is hardware-dependent but generally efficient due to the single buffer binding. The layout is optimized for Vulkan's uniform buffer standards.

**Scaling:**
The layout does not scale with resolution; it is frame-wide. However, the `viewport` field allows for resolution-dependent calculations in shaders.

**Strengths and Limitations:**
Strengths include efficient packing and single-buffer access. Limitations include the fixed size, which may not accommodate additional fields without restructuring.

**When to Use:**
Use this layout for global frame state in voxel rendering pipelines. Avoid for per-instance data.

## Background
The Appearance Frame Uniform Layout originates from the `bend-voxel` project, an unofficial Vulkan-based voxel renderer. It is part of the broader effort to optimize uniform data access in real-time voxel rendering. The layout is defined in `appearance_frame.glsl` and generated via `vulkan/tools/build_shaders.py`.

## Sources
[S1] aivv73/bend-voxel, https://github.com/aivv73/bend-voxel, fetched 2026-10-28 (unofficial)
[S2] Kobakat/VolumetricRayCasting, https://github.com/Kobakat/VolumetricRayCasting, fetched 2026-10-26 (unofficial)
