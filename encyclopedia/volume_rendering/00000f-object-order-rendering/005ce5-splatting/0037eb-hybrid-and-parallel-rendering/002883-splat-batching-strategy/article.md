# Splat Batching Strategy

Also called: Batched Splatting, Gaussian Batching

## Overview
Splat Batching Strategy groups volumetric primitives into discrete batches to reduce draw calls and improve GPU throughput [G]. It addresses the performance bottleneck of processing large numbers of splats individually by leveraging instanced rendering [G]. Unlike [Parallel Splatting](1241-parallel-splatting.md), which distributes work across multiple processing units, this method focuses on optimizing the single-GPU command submission pipeline by packing data efficiently [G].

## How it works
The method transforms a list of individual splat parameters into a structured batch format suitable for GPU consumption [G].

**Inputs and Outputs:**
- Input: A list of $N$ splats, each defined by position, scale, rotation, and opacity [G].
- Output: A batched vertex/index buffer and a single draw call (or a small number of calls) rendering all splats [G].

**Data Structures:**
The core data structure is the "Splat Instance." Each instance represents one splat packed into a fixed-size memory block. The layout for a single splat instance occupies 16 bytes [S2, unofficial]:
- Bytes 0–1: Alpha (opacity), uint8 (0–255), normalized to 0.0–1.0.
- Bytes 2–3: Center X, float16.
- Bytes 4–5: Center Y, float16.
- Bytes 6–7: Center Z, float16.
- Byte 8: Quaternion Octahedral U, uint8.
- Byte 9: Quaternion Octahedral V, uint8.
- Byte 10: Scale X, uint8 (log-encoded).
- Byte 11: Scale Y, uint8 (log-encoded).
- Byte 12: Scale Z, uint8 (log-encoded).
- Byte 13: Quaternion Angle $\theta$, uint8 ($\theta/\pi \cdot 255$).
- Bytes 14–15: Padding.

**Parameters:**
- `SplatInstanceSize`: The number of splats packed into a single instance buffer or draw call. Default value is 128 [S2, unofficial].
- `gaussians_per_splat`: If -1, splats all Gaussians in one call. If positive, splits into batches. Higher values increase speed but raise GPU memory usage [S1, unofficial].

**Steps:**
1. **Pack Splat Data:** Convert each Gaussian's parameters into the 16-byte binary format described above [S2, unofficial].
2. **Allocate Buffers:** Allocate a vertex array of size $4 \times \text{SplatInstanceSize}$ and an index array of size $6 \times \text{SplatInstanceSize}$ [S2, unofficial].
3. **Generate Vertex Positions:** For each instance index $i$ from $0$ to $\text{SplatInstanceSize}-1$:
   - Set four vertex positions at screen-space corners: $(-1,-1)$, $(1,-1)$, $(-1,1)$, $(1,1)$.
   - Set the Z coordinate of these vertices to the value of $i$ cast as a float: `*(float*)&i` [S2, unofficial].
4. **Generate Indices:** For each instance $i$, write two triangles using indices $(0,1,2)$ and $(1,3,2)$ relative to the instance base offset $b = i \times 4$ [S2, unofficial].
5. **Upload and Draw:** Upload the packed splat data and the generated vertex/index buffers to the GPU. Issue a single draw call (e.g., `DrawInstanced` or equivalent) using the batched buffers [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import struct

def batch_splats(splats, splat_instance_size=128):
    """
    Pack splats into batches for GPU rendering.
    splats: List of Gaussian objects with attributes: alpha, center, scale, rotation.
    splat_instance_size: Number of splats per batch (default 128).
    """
    batches = []
    num_splats = len(splats)

    for batch_start in range(0, num_splats, splat_instance_size):
        batch_end = min(batch_start + splat_instance_size, num_splats)
        batch_size = batch_end - batch_start

        # Allocate vertex array: 4 vertices per splat
        vertices = [(0.0, 0.0, 0.0)] * (4 * batch_size)
        # Allocate index array: 6 indices per splat (2 triangles)
        indices = [0] * (6 * batch_size)

        for i in range(batch_size):
            instance_idx = batch_start + i
            splat = splats[instance_idx]

            # Pack splat data into 16-byte structure
            packed_data = pack_splat_to_16bytes(splat)

            # Vertex positions: 4 corners with Z = instance_idx cast to float
            base_offset = i * 4
            vertices[base_offset + 0] = (-1.0, -1.0, float(instance_idx))
            vertices[base_offset + 1] = ( 1.0, -1.0, float(instance_idx))
            vertices[base_offset + 2] = (-1.0,  1.0, float(instance_idx))
            vertices[base_offset + 3] = ( 1.0,  1.0, float(instance_idx))

            # Indices for two triangles: (0,1,2) and (1,3,2) relative to base offset
            indices[base_offset + 0] = base_offset + 0
            indices[base_offset + 1] = base_offset + 1
            indices[base_offset + 2] = base_offset + 2
            indices[base_offset + 3] = base_offset + 1
            indices[base_offset + 4] = base_offset + 3
            indices[base_offset + 5] = base_offset + 2

        batches.append({
            'vertices': vertices,
            'indices': indices,
            'packed_data': packed_data
        })

    return batches

def pack_splat_to_16bytes(splat):
    """
    Pack a single splat into 16 bytes as per [S2, unofficial].
    """
    # Alpha: uint8 0-255
    alpha_byte = int(splat.alpha * 255) & 0xFF

    # Center: float16 (packed as 2 bytes each)
    center_x = struct.pack('<e', splat.center.x)
    center_y = struct.pack('<e', splat.center.y)
    center_z = struct.pack('<e', splat.center.z)

    # Quaternion: Octahedral U, V, Angle
    quat_u = int(splat.rotation.oct_u) & 0xFF
    quat_v = int(splat.rotation.oct_v) & 0xFF
    quat_angle = int((splat.rotation.angle / math.pi) * 255) & 0xFF

    # Scale: Log-encoded uint8
    scale_x = int(splat.scale.x) & 0xFF
    scale_y = int(splat.scale.y) & 0xFF
    scale_z = int(splat.scale.z) & 0xFF

    # Construct 16-byte buffer
    buffer = bytearray(16)
    buffer[0] = alpha_byte
    buffer[1:3] = center_x
    buffer[3:5] = center_y
    buffer[5:7] = center_z
    buffer[8] = quat_u
    buffer[9] = quat_v
    buffer[10] = scale_x
    buffer[11] = scale_y
    buffer[12] = scale_z
    buffer[13] = quat_angle
    # Bytes 14-15 are padding

    return buffer
```

**Edge Cases and Pitfalls:**
- **Instance ID Encoding:** The vertex Z coordinate is set to the instance index cast as a float [S2, unofficial]. This requires the rendering pipeline to handle the Z value appropriately for depth testing [G].
- **Memory Usage:** Increasing `SplatInstanceSize` raises GPU memory usage linearly [S1, unofficial].
- **Precision:** Center coordinates use float16 encoding [S2, unofficial]. Implementers should account for reduced precision in large scenes [G].

## Subtopics

### Splat Instance Mesh Generation
This variant focuses specifically on the geometric representation of each splat as a quad (two triangles) within the batched buffer [S2, unofficial]. It explicitly generates the vertex and index data for each splat instance in CPU memory before upload [S2, unofficial]. It is chosen when fine-grained control over the mesh topology is required [G].

## Operating characteristics
- **Time Complexity:** $O(N)$ for packing and buffer generation, where $N$ is the number of splats. Draw call overhead is reduced from $O(N)$ to $O(N/B)$, where $B$ is the batch size [G].
- **Memory Complexity:** $O(N)$ for storing splat data. Vertex/index buffer size is $O(N)$, specifically $4 \times N$ vertices and $6 \times N$ indices [S2, unofficial].
- **Measured Performance:** Higher values of `gaussians_per_splat` increase speed but raise GPU memory usage [S1, unofficial]. Default `SplatInstanceSize` is 128 [S2, unofficial].
- **Strengths:** Reduces CPU-GPU communication overhead by minimizing draw calls [G].
- **Limitations:** Requires CPU-side buffer generation [G]. Memory usage increases with batch size [S1, unofficial].
- **When to Use:** When rendering large numbers of splats on a single GPU and draw call overhead is a bottleneck [G].
- **When Not to Use:** When VRAM is extremely limited, or when using a rendering pipeline that supports efficient GPU-side instancing without CPU buffer generation [G].

## Background
The technique leverages the efficiency of instanced rendering to handle the massive number of primitives typical in Gaussian Splatting scenes [G]. Specific historical origins and development details are not given in the sources.

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
