# Ray-guided volume rendering with missing brick reporting

Also called: Ray-guided methods

## Overview
Ray-guided volume rendering with missing brick reporting is an out-of-core rendering technique where the working set (the minimal subset of volume data required to render an image) is determined dynamically during the ray-casting process rather than in advance [S1, unofficial][S2]. It solves the problem of memory bottlenecks in large-scale volume rendering by streaming only the bricks accessed by the rays, ensuring optimal accuracy by never loading data that will not be used [S1, unofficial][S2]. It sets itself apart from sibling methods like [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) or [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) by explicitly handling data availability: when a required brick is missing from the GPU cache, the method reports this "missing brick" to a subsystem for subsequent paging, rather than attempting to approximate the data or failing [S2].

## How it works
The method operates on the GPU, performing both ray accumulation and brick identification in parallel to minimize overhead [S2]. It relies on a brick cache (e.g., a 3D texture pool) storing individual bricks of different resolutions in an unordered manner [S1, unofficial]. An additional data structure tracks the residency status of each brick in the volume [S1, unofficial].

**Inputs:**
*   Ray origin and direction [S2].
*   Volume dataset organized into hierarchical bricks (multi-resolution) [S1, unofficial].
*   Current GPU brick cache state [S1, unofficial].
*   Missing brick reporting subsystem (hash table) [S2].

**Outputs:**
*   Accumulated color and opacity for the pixel [S2].
*   Updated missing brick report for the next frame [S2].
*   Resume state (`rayResumePos`, `rayResumeColor`) for interrupted rays [S2].

**Data Structures:**
1.  **Brick Cache:** Unordered collection of bricks in GPU memory, typically a 3D texture pool [S1, unofficial]. Each brick has a unique identifier and associated sampling rates/resolutions [S2].
2.  **Residency Map:** A data structure indicating if a specific brick ID is currently resident in the cache [S1, unofficial].
3.  **Missing Brick Hash Table:** A lock-free, set-based linearly-probed hash table stored in an image load store buffer [S2]. It uses the brick identifier as the hash key to ensure multiple rays requesting the same brick map to the same position, minimizing table size and duplicate entries [S2].

**Algorithm Steps:**
1.  **Initialization:** For each ray, initialize accumulated color `color` to `rayResumeColor` and set boolean flag `terminated` to `true` [S2]. The flag indicates whether a missing brick has been encountered for the current ray in the current frame [S2].
2.  **Ray Loop:** Repeat until the ray exits the volume bounds or saturates (accumulated opacity reaches 1.0) [S2]:
    a.  **Compute Level of Detail (LoD):** Calculate required LoD based on current ray depth [S2].
    b.  **Get Brick:** Retrieve the brick ID and current `samplingRate` available for that brick at the computed LoD [S2]. `GetBrick` always returns some data, potentially a coarser version if the finest is missing [S2].
    c.  **Address Translation:** Use the residency map to translate the sample location from the global reference space (e.g., normalized $[0, 1]^3$) to local texture coordinates within the retrieved brick [S1, unofficial]. Retrieve offsets for the brick from the pool [S2].
    d.  **Check Resolution:** Compare `samplingRate` with `RequiredSamplingForLOD(LoD)` [S2].
    e.  **Report Missing Brick:** If `samplingRate` is not equal to `RequiredSamplingForLOD(LoD)`, the data is undersampled for the desired quality [S2]. Call `ReportMissingBrick(brick)` to add the brick ID to the hash table [S2]. If `terminated` is `true` (first missing brick for this ray), set `terminated` to `false` and store the current ray position in `rayResumePos` [S2]. This allows rendering to resume from this point in the next frame [S2].
    f.  **Raycast:** Perform the raycasting step using the current `ray`, `stepSize`, and `offsets` [S2]. Update `color` and `ray` position.
3.  **Finalization:** Update `rayResumeColor` with the final accumulated `color` [S2]. If the ray completed successfully (did not stop due to a missing brick), set `rayResumePos` to a special `FINISHED` state [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state per ray from previous frame
rayResumeColor = vec4(0.0)
rayResumePos = vec3(0.0)

# Sentinel for completed rays
FINISHED = float('inf')

def RayGuidedRender(ray, volume, cache, hash_table):
    color = rayResumeColor
    terminated = True  # Assume ray will finish unless interrupted

    # Loop until ray exits volume or saturates
    while ray.depth < volume.exit_depth and not ray.saturated:
        # 1. Determine required resolution
        lod = compute_lod(ray.depth)  # Standard: proportional to ray depth

        # 2. Retrieve available brick and its sampling rate
        # Returns best available brick, potentially coarser than requested
        brick_id, sampling_rate = get_brick(ray, lod)

        # 3. Address translation and offset retrieval
        offsets = get_pool_offsets(brick_id)  # Base address/stride from pool

        # 4. Check if data is undersampled
        required_rate = required_sampling_for_lod(lod)

        if sampling_rate != required_rate:
            # Data is missing or too coarse
            hash_table.insert(brick_id)  # Atomic insert for lock-free hash table

            # Record interruption point if this is the first missing brick
            if terminated:
                terminated = False
                rayResumePos = ray.position

        # 5. Perform integration step
        step_size = compute_step_size(ray, lod)
        color = integrate_step(ray, step_size, offsets, color)  # Front-to-back compositing

    # Save state for next frame
    rayResumeColor = color

    if terminated:
        rayResumePos = FINISHED
```

**Parameters:**
*   `rayResumePos`: Stores the position in the volume where the first missing brick occurred, used to resume rendering in subsequent frames [S2].
*   `terminated`: A boolean flag indicating if a missing brick has been encountered for the current ray [S2].
*   `GL_ARB_shader_image_load_store`: The OpenGL extension (core in 4.2) used for atomic operations and independent image buffers to implement the hash table [S2].

**Edge Cases and Pitfalls:**
*   **Undersampling:** If data is undersampled, the renderer displays a coarser version initially but progressively refines regions with finer resolution data until sampled at a single voxel per pixel or the maximum data resolution is reached [S2].
*   **Write Hazards:** The lock-free hash table must handle concurrent writes from multiple rays requesting the same brick, managed via atomic load/store operations [S2].
*   **Spatial Coherency:** Neighboring rays likely request the same bricks, which the hash table exploits by hashing on brick ID to keep the table small and avoid duplicate texture reads back [S2].

## Subtopics

### [Greedy Brick Request Strategy](2843-greedy-brick-request-strategy.md)
The Greedy Brick Request Strategy is a data-loading heuristic used in ray-guided volume rendering to manage out-of-core brick requests. It addresses the problem of missing volume data by requesting all intermediate levels of detail (LoD) along a ray's path when a higher-resolution brick is unavailable, rather than waiting for the final required resolution. Unlike the [Global strategy](2843-global-brick-request-strategy.md), which only requests the final necessary brick, the Greedy strategy floods the request hash table with intermediate bricks. This approach is distinct because it prioritizes iterative visual refinement and faster convergence of the image quality over minimizing the total number of unique brick requests per frame. It produces more pleasing intermediate progress and converges in fewer frames because it allows rays to sample at their final resolution quickly, potentially triggering earlier ray termination.

### Global Brick Request Strategy
The Global Brick Request Strategy is a data-loading heuristic where a ray only requests the bricks absolutely required to satisfy the final rendering request, leaving space in the hash table for other rays [S2]. When a brick is missing, the algorithm increments the Level of Detail (LoD) and looks up the brick at the new resolution, continuing until a missing brick is no longer reported [S2]. Unlike the 'Greedy' strategy, the Global strategy does not request intermediate bricks [S2]. It generates the final correct image quickest in terms of raw completion but may produce less pleasing intermediate progress [S2].

## Operating characteristics
*   **Time Complexity:** The per-frame cost is dominated by the ray casting itself. The overhead of brick identification is minimal as it parallelizes effectively on the GPU [S2]. The hash table operations are $O(1)$ on average due to the lock-free linear probing design [S2].
*   **Memory Complexity:** The working set is minimized to only the bricks accessed during rendering [S1, unofficial]. The hash table size is proportional to the number of unique missing bricks per frame, which is kept small by hashing on brick IDs and exploiting spatial coherency [S2].
*   **Performance:** The method scales to extremely large datasets by streaming data on demand [S2]. It avoids the conservative memory estimates of earlier approaches [S1, unofficial].
*   **Strengths:** Optimal accuracy (never loads unused data), efficient GPU utilization, handles large datasets out-of-core [S2].
*   **Limitations:** Requires multiple frames to converge to full resolution for regions with missing data [S2]. The "Greedy" strategy may generate more hash table entries than necessary, while the "Global" strategy may have slower visual convergence [S2].
*   **When to use:** Large-scale volume rendering where the entire dataset cannot fit in GPU memory and interactive or near-interactive frame rates are required [S2].
*   **When not to use:** Small datasets that fit entirely in GPU memory, or applications requiring single-frame full-resolution accuracy without iterative refinement [S2].

## Background
Ray-guided volume rendering was introduced as a scalable approach to handle large biomedicinal and scientific datasets [S1, unofficial][S2]. Early approaches produced conservative estimates of the working set, leading to sub-optimal memory utilization [S1, unofficial]. The specific method using missing brick reporting with a GPU-based hash table was developed to overcome the limitations of multiple render targets, which were limited in number and ignored spatial coherency [S2]. The technique leverages OpenGL 4.2 features (`GL_ARB_shader_image_load_store`) to implement efficient, lock-free data structures on the GPU [S2]. It represents a shift from pre-computing access patterns to dynamic, ray-driven data streaming [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] An Analysis of Scalable GPU-Based Ray-Guided Volume Rendering, https://pmc.ncbi.nlm.nih.gov/articles/PMC4260263/, fetched 2026-10-26
