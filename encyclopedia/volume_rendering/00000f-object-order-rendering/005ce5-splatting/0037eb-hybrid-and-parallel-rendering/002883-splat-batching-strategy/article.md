# Splat Batching Strategy

Also called: Batched Splatting, Gaussian Batching

## Overview
Splat Batching Strategy is a rendering optimization technique that groups volumetric primitives (typically 3D Gaussians) into discrete batches to reduce the number of draw calls and improve GPU throughput. It addresses the performance bottleneck of processing large numbers of splats individually by leveraging instanced rendering or batched shader execution. Unlike [Parallel Splatting](1241-parallel-splatting.md), which distributes work across multiple processing units, this method focuses on optimizing the single-GPU command submission pipeline by packing data efficiently.

## How it works
The method transforms a list of individual splat parameters into a structured batch format suitable for GPU consumption.

**Inputs and Outputs:**
- Input: A list of $N$ splats, each defined by position, scale, rotation, and opacity.
- Output: A batched vertex/index buffer and a single draw call (or a small number of calls) rendering all splats.

**Data Structures:**
The core data structure is the "Splat Instance." Each instance represents one splat packed into a fixed-size memory block. The layout for a single splat instance (as defined in [S2, unofficial]) is 16 bytes, packed as follows:
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
- Bytes 14–15: Reserved or padding (implied by 16-byte alignment).

**Parameters:**
- `SplatInstanceSize`: The number of splats packed into a single instance buffer or draw call. Default value is 128 [S2, unofficial].
- `gaussians_per_splat`: If -1, splats all Gaussians in one call. If positive, splits into batches. Higher values increase speed but raise GPU memory usage [S1, unofficial].

**Steps:**
1. **Pack Splat Data:** Convert each Gaussian's parameters into the 16-byte binary format described above.
2. **Allocate Buffers:** Allocate a vertex array of size $4 \times \text{SplatInstanceSize}$ and an index array of size $6 \times \text{SplatInstanceSize}$ [S2, unofficial].
3. **Generate Vertex Positions:** For each instance index $i$ from $0$ to $\text{SplatInstanceSize}-1$:
   - Set four vertex positions at screen-space corners: $(-1,-1)$, $(1,-1)$, $(-1,1)$, $(1,1)$.
   - Set the Z coordinate of these vertices to the value of $i$ cast as a float: `*(float*)&i` [S2, unofficial]. This encodes the instance ID into the vertex data for later retrieval in the shader.
4. **Generate Indices:** For each instance $i$, write two triangles using indices $(0,1,2)$ and $(1,3,2)$ relative to the instance base offset $b = i \times 4$ [S2, unofficial].
5. **Upload and Draw:** Upload the packed splat data and the generated vertex/index buffers to the GPU. Issue a single draw call (e.g., `DrawInstanced` or equivalent) using the batched buffers.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
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
        vertices = []
        # Allocate index array: 6 indices per splat (2 triangles)
        indices = []
        
        for i in range(batch_size):
            instance_idx = batch_start + i
            splat = splats[instance_idx]
            
            # Pack splat data into 16-byte structure (pseudo-implementation)
            packed_data = pack_splat_to_16bytes(splat)
            
            # Vertex positions: 4 corners with Z = instance_idx
            # Corner 0: (-1, -1), Z = instance_idx
            # Corner 1: ( 1, -1), Z = instance_idx
            # Corner 2: (-1,  1), Z = instance_idx
            # Corner 3: ( 1,  1), Z = instance_idx
            base_offset = i * 4
            vertices.append((-1, -1, instance_idx))
            vertices.append(( 1, -1, instance_idx))
            vertices.append((-1,  1, instance_idx))
            vertices.append(( 1,  1, instance_idx))
            
            # Indices for two triangles: (0,1,2) and (1,3,2)
            indices.append(base_offset + 0)
            indices.append(base_offset + 1)
            indices.append(base_offset + 2)
            indices.append(base_offset + 1)
            indices.append(base_offset + 3)
            indices.append(base_offset + 2)
            
        batches.append({
            'vertices': vertices,
            'indices': indices,
            'packed_data': packed_data # In practice, this is a separate buffer or interleaved
        })
        
    return batches

def pack_splat_to_16bytes(splat):
    """
    Pack a single splat into 16 bytes as per [S2, unofficial].
    """
    # Alpha: uint8 0-255
    alpha_byte = int(splat.alpha * 255) & 0xFF
    
    # Center: float16
    center_x = pack_float16(splat.center.x)
    center_y = pack_float16(splat.center.y)
    center_z = pack_float16(splat.center.z)
    
    # Quaternion: Octahedral U, V, Angle
    quat_u = pack_oct_u(splat.rotation)
    quat_v = pack_oct_v(splat.rotation)
    quat_angle = int((splat.rotation.angle / math.pi) * 255) & 0xFF
    
    # Scale: Log-encoded uint8
    scale_x = pack_log_scale(splat.scale.x)
    scale_y = pack_log_scale(splat.scale.y)
    scale_z = pack_log_scale(splat.scale.z)
    
    # Construct 16-byte buffer
    # Note: Exact byte order and packing logic depends on GPU shader expectations
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
- **Precision:** Using float16 for center coordinates may introduce precision errors for large scenes. [S2, unofficial] uses float16, so implementers should be aware of potential jitter at extreme scales.
- **Memory Usage:** Increasing `SplatInstanceSize` improves performance but increases GPU memory consumption linearly. If `gaussians_per_splat` is too high, it may exceed VRAM limits [S1, unofficial].
- **Instance ID Encoding:** Encoding the instance ID in the Z coordinate of the vertex assumes that the Z value is not used for depth testing in a standard way. The shader must correctly interpret this Z value as an instance ID rather than a depth value [S2, unofficial].

## Subtopics

### Splat Instance Mesh Generation
This variant focuses specifically on the geometric representation of each splat as a quad (two triangles) within the batched buffer. It sets apart from other batching methods by explicitly generating the vertex/index data for each splat instance in CPU memory before upload, rather than using GPU-side instancing attributes alone. It is chosen when fine-grained control over the mesh topology is required or when the rendering API does not support advanced instancing features. The link to the parent method is direct, as this is the core geometric step of the batching strategy.

## Operating characteristics
- **Time Complexity:** $O(N)$ for packing and buffer generation, where $N$ is the number of splats. Draw call overhead is reduced from $O(N)$ to $O(N/B)$, where $B$ is the batch size.
- **Memory Complexity:** $O(N)$ for storing splat data. Vertex/index buffer size is $O(N)$, specifically $4 \times N$ vertices and $6 \times N$ indices.
- **Measured Performance:** Performance scales with `SplatInstanceSize`. Higher values reduce draw call overhead but increase memory pressure [S1, unofficial]. Default `SplatInstanceSize` of 128 provides a balance [S2, unofficial].
- **Strengths:** Reduces CPU-GPU communication overhead by minimizing draw calls. Simple to implement with standard graphics APIs.
- **Limitations:** Requires CPU-side buffer generation. Memory usage increases with batch size. Precision issues may arise from float16 encoding [S2, unofficial].
- **When to Use:** When rendering large numbers of splats on a single GPU and draw call overhead is a bottleneck.
- **When Not to Use:** When VRAM is extremely limited, or when using a rendering pipeline that supports efficient GPU-side instancing without CPU buffer generation.

## Background
The Splat Batching Strategy is commonly used in real-time 3D Gaussian Splatting implementations, particularly in game engines and interactive applications. It was popularized by libraries such as `gsplat` and `3D-Gaussian-Splatting` for Unity and other platforms [S1, unofficial][S2, unofficial]. The technique leverages the efficiency of instanced rendering to handle the massive number of primitives typical in Gaussian Splatting scenes.

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
