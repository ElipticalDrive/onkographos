# Greedy Meshing with Vertex Pooling

Also called: Greedy Meshing with Vertex Pool, Greedy Meshing with Vertex Pool Bucket System, Voxel Greedy Meshing

## Overview
Greedy Meshing with Vertex Pooling is a rendering optimization technique that combines greedy face merging with a specialized memory management system for vertex buffers and draw calls. It solves the problem of high driver overhead and memory fragmentation when rendering large numbers of small, independent voxel meshes by batching them into shared vertex pools. Unlike standard greedy meshing which outputs individual mesh objects, this method aggregates quads from multiple chunks into contiguous memory buckets, reducing draw calls by an order of magnitude [S1, unofficial]. It sets itself apart from sibling methods by explicitly decoupling the meshing logic from the vertex storage and index generation, using a bucket system to manage memory lifecycle and indirect draw commands [S1, unofficial].

## How it works
The method operates on a per-chunk basis, where each chunk maintains a mesh representation for 6 surface orientations. The core innovation is the use of a `Vertexpool` that manages memory via "buckets" (contiguous blocks of vertices) and an "Indirect Draw Call" (DAIC) structure to track them.

### Inputs and Outputs
*   **Input:** A set of voxel chunks, each containing voxel data (e.g., material IDs or occupancy).
*   **Output:** A single large vertex buffer and an array of DAIC structures, ready for a single draw call (or a few masked draw calls).

### Data Structures
1.  **DAIC (Direct/Indirect Draw Call) Struct:**
    *   `baseVert`: Index of the first vertex in the bucket for this mesh.
    *   `index`: Pointer to the start of the index data in the global index buffer.
    *   `count`: Number of indices (or quads) in this mesh.
    *   `start`: Offset into the index buffer. Set to `0` for negative orientations (-X, -Y, -Z) and `6*K` for positive orientations (+X, +Y, +Z), where `K` is the max quads per bucket [S1, unofficial].
2.  **Vertexpool:**
    *   Manages a pre-allocated memory block.
    *   Maintains a `free` queue of available buckets.
    *   `indirect`: A list/vector of DAIC structs corresponding to active meshes.
3.  **Bucket:**
    *   A contiguous segment of the vertex pool memory.
    *   Initially requested with `max_size` (worst-case capacity).
    *   Filled with quads during meshing.
    *   Size reduced (logical count) after meshing is complete.

### Algorithm Steps

1.  **Initialization:**
    *   Allocate a global vertex buffer and index buffer.
    *   Initialize the `Vertexpool` with a fixed maximum mapped buffer size. This size cannot be exceeded later [S1, unofficial].
    *   For each chunk, initialize 6 DAIC pointers (one per orientation).

2.  **Meshing Loop (Per Orientation):**
    *   Iterate through each of the 6 surface orientations (+X, -X, +Y, -Y, +Z, -Z) separately.
    *   **Request Bucket:** Request a new bucket from the `Vertexpool` with `max_size`.
    *   **Assign Group:** Assign the current orientation's mesh to this bucket.
    *   **Generate Quads:** Run the greedy meshing algorithm for this orientation. For each quad generated:
        *   Add its 4 vertices to the current bucket.
        *   Store the quad's index offset relative to the bucket's start.
    *   **Finalize Bucket:** After all quads for this orientation are computed, reduce the bucket's logical size to the actual number of quads filled.

3.  **Index Buffer Construction:**
    *   The index buffer is shared. Indices are constructed based on orientation to allow reusing the same index pattern for different winding orders.
    *   Let `K` be the number of quads in the current bucket.
    *   **Negative Orientations (-X, -Y, -Z):**
        *   For each quad `j` from `0` to `K-1`:
            *   Triangle 1: `j*4+0`, `j*4+1`, `j*4+2`
            *   Triangle 2: `j*4+3`, `j*4+1`, `j*4+0`
    *   **Positive Orientations (+X, +Y, +Z):**
        *   For each quad `j` from `0` to `K-1`:
            *   Triangle 1: `j*4+0`, `j*4+2`, `j*4+1`
            *   Triangle 2: `j*4+1`, `j*4+3`, `j*4+0`
    *   Set the `start` property in the DAIC struct: `0` for negative, `6*K` for positive [S1, unofficial].

4.  **Draw Execution:**
    *   Use the DAIC array to issue draw calls.
    *   Optionally, mask and sort DAICs for occlusion culling or alpha blending [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VertexPool:
    def __init__(self, max_buffer_size):
        self.buffer = allocate_memory(max_buffer_size)
        self.free_queue = [bucket for bucket in range(max_buffer_size // bucket_size)]
        self.indirect_draw_calls = []

    def request_bucket(self, max_quads):
        if not self.free_queue:
            raise MemoryError("Vertex pool exhausted")
        bucket_idx = self.free_queue.pop(0)
        return Bucket(self.buffer, bucket_idx, max_quads)

    def unsection(self, bucket_idx):
        # Deconstruct vertices is optional [G]
        # Add bucket to free queue
        self.free_queue.append(bucket_idx)
        # Remove from indirect list
        # Note: Implementation details vary, but logically the DAIC is removed
        pass

class Chunk:
    def __init__(self, size):
        self.size = size
        self.daic_pointers = [None] * 6  # +X, -X, +Y, -Y, +Z, -Z

    def mesh_and_pool(self, voxel_data, vertex_pool):
        orientations = ['+X', '-X', '+Y', '-Y', '+Z', '-Z']
        
        for i, orientation in enumerate(orientations):
            # 1. Request bucket
            # Calculate worst-case quads for this orientation
            # max_quad = (length^3 * 6 * (1-x) * x) where x is occupancy fraction
            # In practice, use a safe upper bound or dynamic resizing if supported
            bucket = vertex_pool.request_bucket(max_quads=10000) 
            
            # 2. Run Greedy Meshing for this orientation
            quads = greedy_mesh(voxel_data, orientation)
            
            # 3. Fill bucket
            for quad in quads:
                bucket.add_vertices(quad.vertices)
            
            # 4. Create DAIC
            daic = DAIC()
            daic.base_vert = bucket.start_index
            daic.count = len(quads)
            
            # 5. Set index offset
            if orientation in ['-X', '-Y', '-Z']:
                daic.start = 0
            else:
                daic.start = 6 * len(quads) # Offset for positive winding
            
            self.daic_pointers[i] = daic
            vertex_pool.indirect_draw_calls.append(daic)
            
            # Note: Bucket is not freed yet, it is part of the mesh

    def finalize(self, vertex_pool):
        # After all chunks are processed, if using a single pool for the whole frame
        # or if buckets need to be reused, unsection them.
        # In a static scene, buckets might persist.
        # In dynamic, unsection after draw.
        pass

def render(vertex_pool, daic_list):
    # Issue draw calls using DAIC list
    # Optional: Mask and sort daic_list
    pass
```

### Parameters and Typical Values
*   **Chunk Size:** 16x16x16 or 32x32x32 voxels [S1, unofficial].
*   **Bucket Size:** Should correspond to the maximum number of meshes or sub-meshes to be drawn simultaneously. A single bucket size is often used for simplicity [S1, unofficial].
*   **Noise Parameters (for testing):** OpenSimplex2, frequency 1 over visible region, 8 octaves, lacunarity 2.0, gain 0.6 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Memory Leak:** If occupied buckets are not un-sectioned after utilization, a memory leak occurs. This must be handled in the `Vertexpool` destructor or after frame completion [S1, unofficial].
*   **Buffer Overflow:** The original mapped buffer size cannot be exceeded, even if the vertex pool is cleared and bucket sizes adjusted. Plan for worst-case occupancy [S1, unofficial].
*   **Vertex Deconstruction:** Deconstructing vertices in a bucket is not strictly necessary as the bucket is freed and will be overwritten on re-sectioning. Discretion depends on the `Vertex` struct's constructor [S1, unofficial].
*   **Index Buffer Limits:** The index buffer must be large enough to hold indices for all orientations. The offset logic (`start` property) ensures correct indexing for different winding orders [S1, unofficial].

## Subtopics

### Greedy Meshing with Vertex Pool Bucketing
A variant that emphasizes the bucketing mechanism for memory management. It highlights that deconstructing vertices is optional since buckets are overwritten upon re-use. The choice to deconstruct depends on the `Vertex` struct's constructor requirements [S1, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   **Meshing:** Identical to standard greedy meshing, $O(N)$ where $N$ is the number of voxels.
    *   **Draw Calls:** Reduced from $O(M)$ (where $M$ is the number of chunks) to $O(1)$ or $O(K)$ (where $K$ is the number of buckets/orientations).
    *   **Masking/Ordering:** $O(K \log K)$ or $O(K)$ depending on sorting algorithm. Measured times: 160us for $3^3$ chunks, 260us for $5^5$ chunks, 1800us for $10^3$ chunks [S1, unofficial].
*   **Memory Complexity:**
    *   Vertex buffer size is proportional to the total number of quads across all chunks.
    *   Index buffer size is proportional to the total number of quads.
    *   Bucket overhead is minimal if bucket sizes are well-matched.
*   **Performance:**
    *   **Static Rendering:** Vertex pool is consistently almost twice as fast as naive meshing. For 16x16x16 chunks, naive draw times ranged from 4.9ms to 170ms, while pool draw times ranged from 3.9ms to 91ms [S1, unofficial].
    *   **Dynamic Rendering:** Vertex pool outperforms naive implementation even with frequent remeshing. For 16x16x16 chunks, pool mesh times were 521us-950us, while naive mesh times were 380us-746us. The pool reduces driver overhead [S1, unofficial].
    *   **Scaling:** The naive version crashes for large scenes (e.g., 10x10x10 chunks of 32x32x32 voxels) due to driver limits, while the vertex pool handles them (186ms) [S1, unofficial].
    *   **Over-bucketing:** Under-filling buckets has no significant effect on rendering or meshing times (e.g., K=2400 vs K=9600 for ~2200 quads) [S1, unofficial].
*   **Strengths:**
    *   Drastically reduces draw calls.
    *   Mitigates driver overhead.
    *   Handles large scenes that would crash naive implementations.
    *   Allows for easy masking and sorting for occlusion culling and alpha blending [S1, unofficial].
*   **Limitations:**
    *   Does not optimize vertex data specifics or draw primitives; still requires a meshing step.
    *   Requires careful memory management to avoid leaks.
    *   Single bucket size may lead to memory waste if not tuned [S1, unofficial].

## Background
This method was developed by Nick McDermott as part of his "High Performance Voxel Engine" project in 2021. It was designed to address the performance bottlenecks of rendering large voxel worlds with many small meshes. The technique builds on [Greedy Meshing](1776-greedy-meshing.md) by adding a layer of memory management abstraction to optimize the rendering pipeline [S1, unofficial].

## Sources
[S1] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
