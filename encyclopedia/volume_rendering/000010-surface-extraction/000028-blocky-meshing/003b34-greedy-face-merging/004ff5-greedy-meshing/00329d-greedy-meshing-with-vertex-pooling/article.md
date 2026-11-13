# Greedy Meshing with Vertex Pooling

Also called: Greedy Meshing with Vertex Pool, Greedy Meshing with Vertex Pool Bucket System, Voxel Greedy Meshing

## Overview
Greedy Meshing with Vertex Pooling is a rendering optimization technique that combines greedy face merging with a specialized memory management system for vertex buffers and draw calls [S1, unofficial]. It solves the problem of high driver overhead and memory fragmentation when rendering large numbers of small, independent voxel meshes by batching them into shared vertex pools [S1, unofficial]. Unlike standard greedy meshing which outputs individual mesh objects, this method aggregates quads from multiple chunks into contiguous memory buckets, reducing draw calls by an order of magnitude [S1, unofficial]. It sets itself apart from sibling methods by explicitly decoupling the meshing logic from the vertex storage and index generation, using a bucket system to manage memory lifecycle and indirect draw commands [S1, unofficial].

## How it works
The method operates on a per-chunk basis, where each chunk maintains a mesh representation for 6 surface orientations [S1, unofficial]. The core innovation is the use of a `Vertexpool` that manages memory via "buckets" (contiguous blocks of vertices) and an "Indirect Draw Call" (DAIC) structure to track them [S1, unofficial].

### Inputs and Outputs
*   **Input:** A set of voxel chunks, each containing voxel data (e.g., material IDs or occupancy) [S1, unofficial].
*   **Output:** A single large vertex buffer and an array of DAIC structures, ready for draw calls [S1, unofficial].

### Data Structures
1.  **DAIC (Direct/Indirect Draw Call) Struct:**
    *   `baseVert`: Index of the first vertex in the bucket for this mesh [S1, unofficial].
    *   `index`: Pointer to the start of the index data in the global index buffer [S1, unofficial].
    *   `count`: Number of indices (or quads) in this mesh [S1, unofficial].
    *   `start`: Offset into the index buffer. Set to `0` for negative orientations (-X, -Y, -Z) and `6*K` for positive orientations (+X, +Y, +Z), where `K` is the max quads per bucket [S1, unofficial].
2.  **Vertexpool:**
    *   Manages a pre-allocated memory block [S1, unofficial].
    *   Maintains a `free` queue of available buckets [S1, unofficial].
    *   `indirect`: A list/vector of DAIC structs corresponding to active meshes [S1, unofficial].
3.  **Bucket:**
    *   A contiguous segment of the vertex pool memory [S1, unofficial].
    *   Initially requested with `max_size` (worst-case capacity) [S1, unofficial].
    *   Filled with quads during meshing [S1, unofficial].
    *   Size reduced (logical count) after meshing is complete [S1, unofficial].

### Algorithm Steps
1.  **Initialization:**
    *   Allocate a global vertex buffer and index buffer [S1, unofficial].
    *   Initialize the `Vertexpool` with a fixed maximum mapped buffer size. This size cannot be exceeded later [S1, unofficial].
    *   For each chunk, initialize 6 DAIC pointers (one per orientation) [S1, unofficial].
2.  **Meshing Loop (Per Orientation):**
    *   Iterate through each of the 6 surface orientations (+X, -X, +Y, -Y, +Z, -Z) separately [S1, unofficial].
    *   **Request Bucket:** Request a new bucket from the `Vertexpool` with `max_size` [S1, unofficial].
    *   **Assign Group:** Assign the current orientation's mesh to this bucket [S1, unofficial].
    *   **Generate Quads:** Run the greedy meshing algorithm for this orientation. For each quad generated:
        *   Add its 4 vertices to the current bucket [S1, unofficial].
        *   Store the quad's index offset relative to the bucket's start [S1, unofficial].
    *   **Finalize Bucket:** After all quads for this orientation are computed, reduce the bucket's logical size to the actual number of quads filled [S1, unofficial].
3.  **Index Buffer Construction:**
    *   The index buffer is shared. Indices are constructed based on orientation to allow reusing the same index pattern for different winding orders [S1, unofficial].
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
    *   Use the DAIC array to issue draw calls [S1, unofficial].
    *   Optionally, mask and sort DAICs for occlusion culling [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class DAIC:
    def __init__(self):
        self.baseVert = 0
        self.index = 0
        self.count = 0
        self.start = 0

class Bucket:
    def __init__(self, start_idx, max_quads):
        self.start_idx = start_idx
        self.max_quads = max_quads
        self.current_quads = 0

    def add_quad(self):
        self.current_quads += 1

    def finalize(self):
        return self.current_quads

class VertexPool:
    def __init__(self, max_buffer_size, vertex_size):
        self.max_buffer_size = max_buffer_size
        self.vertex_size = vertex_size
        self.free_queue = []
        self.indirect_draw_calls = []
        self.next_free_idx = 0

    def request_bucket(self, max_quads):
        if not self.free_queue:
            if self.next_free_idx * max_quads * self.vertex_size > self.max_buffer_size:
                raise MemoryError("Vertex pool exhausted")
            bucket = Bucket(self.next_free_idx * max_quads, max_quads)
            self.next_free_idx += 1
            return bucket
        bucket_idx = self.free_queue.pop(0)
        return Bucket(bucket_idx, max_quads)

    def unsection(self, bucket):
        self.free_queue.append(bucket.start_idx)
        # Remove from indirect list if needed
        pass

def construct_indices(negative_quads, positive_quads):
    indices = []
    # Negative orientations (-X, -Y, -Z)
    for j in range(negative_quads):
        indices.extend([j*4+0, j*4+1, j*4+2])  # Triangle 1
        indices.extend([j*4+3, j*4+1, j*4+0])  # Triangle 2
    # Positive orientations (+X, +Y, +Z)
    for j in range(positive_quads):
        indices.extend([j*4+0, j*4+2, j*4+1])  # Triangle 1
        indices.extend([j*4+1, j*4+3, j*4+0])  # Triangle 2
    return indices

def mesh_chunk(chunk_data, vertex_pool, chunk_length):
    orientations = ['+X', '-X', '+Y', '-Y', '+Z', '-Z']
    daic_pointers = [None] * 6
    occupancy_fraction = 0.5  # Example value
    max_quad = (chunk_length**3 * 6 * (1 - occupancy_fraction) * occupancy_fraction)

    for i, orientation in enumerate(orientations):
        bucket = vertex_pool.request_bucket(int(max_quad))
        # [G] Calls parent Greedy Meshing algorithm for this orientation
        quads = greedy_mesh(chunk_data, orientation)
        for _ in quads:
            bucket.add_quad()
        actual_quads = bucket.finalize()

        daic = DAIC()
        daic.baseVert = bucket.start_idx
        daic.count = actual_quads
        if orientation in ['-X', '-Y', '-Z']:
            daic.start = 0
        else:
            daic.start = 6 * actual_quads
        daic_pointers[i] = daic
        vertex_pool.indirect_draw_calls.append(daic)
    return daic_pointers
```

### Parameters and Typical Values
*   **Chunk Size:** 16x16x16 or 32x32x32 voxels [S1, unofficial].
*   **Bucket Size:** Should correspond to the maximum number of meshes or sub-meshes to be drawn simultaneously. A single bucket size is often used for simplicity [S1, unofficial].
*   **Noise Parameters (for testing):** OpenSimplex2, frequency 1 over visible region, 8 octaves, lacunarity 2.0, gain 0.6 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Memory Leak:** If occupied buckets are not un-sectioned after utilization, a memory leak occurs. This must be handled in the `Vertexpool` destructor or after frame completion [S1, unofficial].
*   **Buffer Overflow:** The original mapped buffer size cannot be exceeded, even if the vertex pool is cleared and bucket sizes adjusted [S1, unofficial].
*   **Vertex Deconstruction:** Deconstructing vertices in a bucket is not strictly necessary as the bucket is freed and will be overwritten on re-sectioning. Discretion depends on the `Vertex` struct's constructor [S1, unofficial].
*   **Index Buffer Limits:** The index buffer must be large enough to hold indices for all orientations. The offset logic (`start` property) ensures correct indexing for different winding orders [S1, unofficial].

## Subtopics

### Greedy Meshing with Vertex Pool Bucketing
A variant that emphasizes the bucketing mechanism for memory management. It highlights that deconstructing vertices is optional since buckets are overwritten upon re-use. The choice to deconstruct depends on the `Vertex` struct's constructor requirements [S1, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   **Meshing:** Identical to standard greedy meshing, $O(N)$ where $N$ is the number of voxels [S1, unofficial].
    *   **Draw Calls:** Reduced from $O(M)$ (where $M$ is the number of chunks) to $O(1)$ or $O(K)$ (where $K$ is the number of buckets/orientations) [S1, unofficial].
    *   **Masking/Ordering:** $O(K \log K)$ or $O(K)$ depending on sorting algorithm. Measured times: 160us for $3^3$ chunks, 260us for $5^3$ chunks, 1800us for $10^3$ chunks [S1, unofficial].
*   **Memory Complexity:**
    *   Vertex buffer size is proportional to the total number of quads across all chunks [S1, unofficial].
    *   Index buffer size is proportional to the total number of quads [S1, unofficial].
    *   Bucket overhead is minimal if bucket sizes are well-matched [S1, unofficial].
*   **Performance:**
    *   **Static Rendering:** Vertex pool is consistently almost twice as fast as naive meshing. For 16x16x16 chunks, naive draw times were 4.9ms, 11ms, 64ms, and 170ms for 3x3x3, 5x5x5, 10x10x10, and 15x15x15 chunk grids respectively, while pool draw times were 3.9ms, 6.1ms, 27ms, and 91ms [S1, unofficial].
    *   **Dynamic Rendering:** Vertex pool outperforms naive implementation even with frequent remeshing. For 16x16x16 chunks, pool mesh times were 521us-950us, while naive mesh times were 380us-746us. The pool reduces driver overhead [S1, unofficial].
    *   **Scaling:** The naive version crashes for large scenes (e.g., 10x10x10 chunks of 32x32x32 voxels) due to driver limits, while the vertex pool handles them (186ms) [S1, unofficial].
    *   **Over-bucketing:** Under-filling buckets has no significant effect on rendering or meshing times (e.g., K=2400 vs K=9600 for ~2200 quads) [S1, unofficial].
*   **Strengths:**
    *   Drastically reduces draw calls [S1, unofficial].
    *   Mitigates driver overhead [S1, unofficial].
    *   Handles large scenes that would crash naive implementations [S1, unofficial].
    *   Allows for masking and sorting for occlusion culling [S1, unofficial].
*   **Limitations:**
    *   Does not optimize vertex data specifics or draw primitives; still requires a meshing step [S1, unofficial].
    *   Requires careful memory management to avoid leaks [S1, unofficial].
    *   Single bucket size may lead to memory waste if not tuned [S1, unofficial].

## Background
This method was developed by Nick McDermott as part of his "High Performance Voxel Engine" project in 2021. It was designed to address the performance bottlenecks of rendering large voxel worlds with many small meshes. The technique builds on [Greedy Meshing](3519-greedy-meshing.md) by adding a layer of memory management abstraction to optimize the rendering pipeline [S1, unofficial].

## Sources
[S1] High Performance Voxel Engine (2021), https://nickmcd.me/2021/04/04/high-performance-voxel-engine/, fetched 2026-10-26 (unofficial)
