# Splat Batching Strategy

Also called: Batched Splatting, Gaussian Batching

## Overview
Splat Batching Strategy reduces draw call overhead in 3D Gaussian Splatting by grouping Gaussians into fixed-size batches for instanced rendering [S1, unofficial]. A single parameter, `gaussians_per_splat`, controls the batch size: if set to -1, all Gaussians are rendered in one call; if positive, they are split into batches of that size [S1, unofficial]. Higher values increase rendering speed but raise GPU memory usage [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A list of $N$ 3D Gaussians with position, scale, rotation, opacity, and spherical harmonics coefficients [G].
- **Output:** A rendered image where overlapping splats are composited [G].
- **Parameter:** `gaussians_per_splat` (also called `SplatInstanceSize`). Default value is 128 [S2, unofficial].

**Data Structures:**
1. **Vertex Buffer:** An array of size $4 \times \text{SplatInstanceSize}$ [S2, unofficial].
2. **Index Buffer:** An array of size $6 \times \text{SplatInstanceSize}$ [S2, unofficial].
3. **Instance Buffer:** Per-instance attributes packed according to the compression scheme [S2, unofficial]. The Spark layout uses: position as float16 $(x,y,z)$, quaternion as octahedral coordinates $(U,V)$ plus angle $\theta$, scale as log-encoded uint8 $(x,y,z)$, opacity as uint8 (0–255 mapped to 0.0–1.0), and spherical harmonics as packed uint32 arrays [S2, unofficial].

**Steps:**
1. **Calculate Batches:** Determine the number of batches $B = \lceil N / \text{SplatInstanceSize} \rceil$ [G].
2. **Allocate Buffers:** Allocate vertex and index buffers of size $4 \times \text{SplatInstanceSize}$ and $6 \times \text{SplatInstanceSize}$ respectively [S2, unofficial].
3. **Generate Geometry:** For each instance index $i$ from $0$ to $\text{SplatInstanceSize}-1$, set four vertex positions at screen-space corners $(-1,-1), (1,-1), (-1,1), (1,1)$ with the Z coordinate cast from $i$ as `*(float*)&i` [S2, unofficial]. Write two triangles using indices $(0,1,2)$ and $(1,3,2)$ relative to the instance base offset $b = i \times 4$ [S2, unofficial].
4. **Populate Instance Data:** Fill the instance buffer with the packed attributes for the current batch of Gaussians [S2, unofficial].
5. **Render Loop:** For each batch, bind the vertex, index, and instance buffers, then issue an instanced draw call with `SplatInstanceSize` instances [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def pack_gaussian(g):
    # Returns packed attributes matching the Spark layout [S2, unofficial]
    pos = (g.x, g.y, g.z)  # float16
    quat = (g.quat_u, g.quat_v, g.quat_theta)  # uint8
    scale = (g.scale_x, g.scale_y, g.scale_z)  # uint8 log-encoded
    opacity = g.opacity  # uint8 0-255
    sh = g.sh_coeffs  # packed uint32
    return (pos, quat, scale, opacity, sh)

def render_splats(gaussians, gaussians_per_splat):
    N = len(gaussians)
    if gaussians_per_splat == -1:
        gaussians_per_splat = N
    
    num_batches = (N + gaussians_per_splat - 1) // gaussians_per_splat
    
    for b in range(num_batches):
        start_idx = b * gaussians_per_splat
        end_idx = min(start_idx + gaussians_per_splat, N)
        current_batch_size = end_idx - start_idx
        
        vertices = []
        indices = []
        
        for i in range(current_batch_size):
            instance_id = start_idx + i
            # Z coordinate is a bit-cast of the integer index [S2, unofficial]
            z = struct.unpack('f', struct.pack('I', instance_id))[0]
            
            vertices.extend([(-1, -1, z), (1, -1, z), (-1, 1, z), (1, 1, z)])
            
            base_offset = i * 4
            indices.extend([base_offset, base_offset + 1, base_offset + 2])
            indices.extend([base_offset + 1, base_offset + 3, base_offset + 2])
            
        instance_data = [pack_gaussian(gaussians[start_idx + i]) for i in range(current_batch_size)]
        
        # Issue instanced draw call with current_batch_size instances
        # draw_instanced(vertices, indices, instance_data, current_batch_size)
```

**Edge Cases and Pitfalls:**
- **Last Batch Padding:** If `gaussians_per_splat` exceeds the remaining Gaussians in the final batch, the draw call must be adjusted to render only `current_batch_size` instances rather than the full buffer size [G].

## Subtopics

### Splat Instance Mesh Generation
This variant defines the vertex and index layout required for instanced splat rendering [S2, unofficial]. Each instance represents a single Gaussian, and the mesh consists of two triangles forming a quad [S2, unofficial]. The vertices are defined in screen space with corners at $(-1,-1), (1,-1), (-1,1), (1,1)$, and the Z coordinate is set to the instance index cast as a float to pass instance data through the rendering pipeline [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ total work, with draw calls reduced to $O(N / \text{SplatInstanceSize})$ [S1, unofficial].
- **Memory Complexity:** $O(\text{SplatInstanceSize})$ for vertex, index, and instance buffers per batch [S1, unofficial].
- **Performance:** Reduces CPU-GPU synchronization overhead by minimizing driver calls [S1, unofficial]. Higher batch sizes improve throughput but increase memory consumption [S1, unofficial].
- **Strengths:** Efficiently handles scenes with tens or hundreds of thousands of splats by amortizing draw call overhead [S1, unofficial].
- **Limitations:** Requires careful batch sizing to balance memory pressure and rendering speed [S1, unofficial].

## Background
Splat Batching Strategy is a rendering optimization technique applied within 3D Gaussian Splatting pipelines to improve real-time performance [S1, unofficial]. The specific instancing geometry, buffer layouts, and compression formats are documented in community implementations [S2, unofficial].

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
