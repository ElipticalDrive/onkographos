# Splat Batching Strategy

Also called: Batched Splatting, Gaussian Batching

## Overview
Splat Batching Strategy is a GPU optimization technique for 3D Gaussian Splatting that reduces draw call overhead by packing multiple Gaussian splats into a single instanced rendering command. It addresses the performance bottleneck of issuing individual draw calls for each Gaussian, which is inefficient when rendering scenes with tens or hundreds of thousands of splats. Unlike non-tiled approaches that may process all splats in one massive batch or tile-based methods that split work spatially, batching focuses on temporal efficiency by grouping splats into fixed-size chunks for instanced drawing.

## How it works
The method relies on GPU instancing to render multiple geometric primitives (splat quads) with a single draw call. Each instance corresponds to one 3D Gaussian, transformed into screen space.

**Inputs and Outputs:**
- **Input:** A list of $N$ 3D Gaussians, each defined by position, covariance (or scale/rotation), opacity, and spherical harmonics coefficients.
- **Output:** A rendered image where each pixel accumulates color from overlapping splats.
- **Parameter:** `gaussians_per_splat` (or `SplatInstanceSize`). If set to -1, all Gaussians are rendered in a single batch. If positive, the Gaussians are split into batches of this size. Higher values increase throughput but raise GPU memory usage for vertex/index buffers [S1, S2, unofficial]. Default value is 128 [S2, unofficial].

**Data Structures:**
1.  **Vertex Buffer:** An array of size $4 \times \text{SplatInstanceSize}$. Each instance contributes 4 vertices representing the corners of a screen-space quad.
2.  **Index Buffer:** An array of size $6 \times \text{SplatInstanceSize}$. Each instance contributes 2 triangles (6 indices).
3.  **Instance Buffer:** A buffer containing per-instance attributes (position, scale, rotation, opacity, SH coefficients). The layout depends on the specific compression format (e.g., Spark compression uses packed uint8/uint16 fields for position, quaternion, and scale) [S2, unofficial].

**Steps:**
1.  **Calculate Batches:** Determine the number of batches $B = \lceil N / \text{SplatInstanceSize} \rceil$.
2.  **Allocate Buffers:** For each batch, allocate vertex and index buffers of size $4 \times \text{SplatInstanceSize}$ and $6 \times \text{SplatInstanceSize}$ respectively [S2, unofficial].
3.  **Generate Geometry:**
    - For each instance index $i$ from $0$ to $\text{SplatInstanceSize}-1$:
        - Set four vertex positions at screen-space corners: $(-1,-1), (1,-1), (-1,1), (1,1)$.
        - Set the Z coordinate of these vertices to the instance index $i$ cast as a float: $z = \text{float}(i)$ [S2, unofficial]. This allows the fragment shader to identify the instance ID.
        - Write two triangles using indices $(0,1,2)$ and $(1,3,2)$ relative to the instance base offset $b = i \times 4$ [S2, unofficial].
4.  **Populate Instance Data:** Fill the instance buffer with the attributes for the current batch of Gaussians. Attributes typically include:
    - Center $(x, y, z)$ encoded as float16 or uint8 [S2, unofficial].
    - Quaternion encoded as octahedral coordinates (U, V) and angle $\theta$ [S2, unofficial].
    - Scale $(x, y, z)$ log-encoded to uint8 [S2, unofficial].
    - Opacity as uint8 (0–255 mapped to 0.0–1.0) [S2, unofficial].
    - Spherical Harmonics coefficients packed into uint32 arrays [S2, unofficial].
5.  **Render Loop:** For each batch $b$ from $0$ to $B-1$:
    - Bind the vertex, index, and instance buffers for the current batch.
    - Issue an instanced draw call (e.g., `glDrawElementsInstanced`) with `SplatInstanceSize` instances.
    - The vertex shader transforms the screen-space quad into world/view space using the instance attributes.
    - The fragment shader computes the 2D Gaussian contribution and blends it into the frame buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_splats(gaussians, gaussians_per_splat):
    N = len(gaussians)
    if gaussians_per_splat == -1:
        gaussians_per_splat = N
    
    num_batches = (N + gaussians_per_splat - 1) // gaussians_per_splat
    
    for b in range(num_batches):
        start_idx = b * gaussians_per_splat
        end_idx = min(start_idx + gaussians_per_splat, N)
        current_batch_size = end_idx - start_idx
        
        # Allocate buffers
        # Vertex buffer: 4 vertices per instance
        vertices = []
        for i in range(current_batch_size):
            instance_id = start_idx + i
            # Screen space corners with Z = instance_id
            vertices.append((-1, -1, float(instance_id)))
            vertices.append(( 1, -1, float(instance_id)))
            vertices.append((-1,  1, float(instance_id)))
            vertices.append(( 1,  1, float(instance_id)))
            
        # Index buffer: 2 triangles per instance (6 indices)
        indices = []
        for i in range(current_batch_size):
            base_offset = i * 4
            indices.extend([base_offset, base_offset+1, base_offset+2])
            indices.extend([base_offset+1, base_offset+3, base_offset+2])
            
        # Prepare instance data (attributes for each Gaussian in this batch)
        instance_data = []
        for i in range(current_batch_size):
            g = gaussians[start_idx + i]
            # Encode attributes according to compression scheme (e.g., Spark)
            # Position: float16 x,y,z
            # Quaternion: octahedral U, V, angle theta
            # Scale: log-encoded uint8 x,y,z
            # Opacity: uint8
            # SH: packed uint32
            instance_data.append(encode_gaussian(g))
            
        # Issue instanced draw call
        # glDrawElementsInstanced(GL_TRIANGLES, 6 * current_batch_size, GL_UNSIGNED_INT, 0, current_batch_size)
        # Note: In practice, indices are often static or generated per-batch.
        # The vertex shader uses the Z coordinate to retrieve instance attributes.
```

**Edge Cases and Pitfalls:**
- **Precision:** Using the instance index as the Z-coordinate in the vertex shader requires sufficient precision. If the number of instances exceeds the range of the data type used for Z, artifacts may occur [S2, unofficial].
- **Memory Usage:** Large batch sizes increase the size of the vertex and index buffers. If the batch size is too large, it may exceed GPU memory limits or cause cache thrashing [S1, unofficial].
- **Degenerate Input:** If `gaussians_per_splat` is larger than the number of remaining Gaussians in the last batch, the buffer should be padded or the draw call adjusted to only render the valid instances [G].

## Subtopics

### Splat Instance Mesh Generation
This variant focuses on the specific geometry generation for instanced rendering. It defines the vertex and index layout for the splat quads. Each instance represents a single Gaussian, and the mesh consists of two triangles forming a quad. The vertices are defined in screen space with corners at $(-1,-1), (1,-1), (-1,1), (1,1)$, and the Z coordinate is set to the instance index to allow the fragment shader to identify the instance. This approach simplifies the vertex shader by avoiding complex transformations for each vertex, instead relying on the fragment shader to compute the Gaussian contribution based on the instance ID [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of Gaussians. The number of draw calls is reduced to $O(N / \text{SplatInstanceSize})$.
- **Memory Complexity:** $O(\text{SplatInstanceSize})$ for the vertex and index buffers per batch. The instance buffer size is also proportional to the batch size.
- **Performance:** Reduces CPU-GPU synchronization overhead by minimizing draw calls. Higher batch sizes improve throughput but increase memory usage [S1, unofficial].
- **Strengths:** Efficient for scenes with many Gaussians; reduces driver overhead.
- **Limitations:** Requires careful management of batch sizes to balance memory and performance; may not be suitable for dynamic scenes with frequent updates if buffer updates are expensive.

## Background
Splat Batching Strategy is derived from the broader field of 3D Gaussian Splatting, which was introduced by Kerbl et al. in 2023. The specific implementation details, such as the use of screen-space quads with instance indices in the Z-coordinate and the Spark compression format, are documented in unofficial repositories like `gsplat-unity` and `3D-Gaussian-Splatting` [S1, S2, unofficial]. These implementations aim to optimize the rendering pipeline for real-time applications on GPUs.

## Sources
[S1] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S2] wuyize25/gsplat-unity, https://github.com/wuyize25/gsplat-unity, fetched 2026-10-27 (unofficial)
