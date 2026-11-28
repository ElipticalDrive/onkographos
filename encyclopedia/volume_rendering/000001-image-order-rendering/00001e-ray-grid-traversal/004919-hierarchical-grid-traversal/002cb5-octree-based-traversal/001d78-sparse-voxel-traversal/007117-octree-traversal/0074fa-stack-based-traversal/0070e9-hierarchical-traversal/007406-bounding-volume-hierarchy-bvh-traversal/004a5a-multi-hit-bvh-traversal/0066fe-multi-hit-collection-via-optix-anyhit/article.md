# Multi-hit Collection via OptiX anyhit

Also called: Collecting all intersections using anyhit in OptiX, Multiple-hit collection via anyhit, Tetrahedral volume rendering intersection collection

## Overview
A technique to process all intersections along a ray for volume rendering on a tetrahedral mesh by modifying a payload in the anyhit shader and collecting results in the raygen program, avoiding the overhead of multiple `optixTrace` calls [S1, unofficial]. It sets itself apart from multiple closehit calls by shaving off per-ray-cast overhead at the cost of memory bandwidth for storing hits and explicit sorting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin, direction, `tmin`, `tmax`, and a traversable acceleration structure.
*   **Output:** A sorted list of intersection payloads (e.g., tetrahedron ID, barycentric coordinates, depth) in the raygen program.

**Data Structures:**
The payload structure must be passed via `optixSetPayload` and retrieved via `optixGetPayload`. A typical structure for tetrahedral volume rendering includes:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Payload {
    float4 barycentric; // Barycentric coordinate of the intersection
    int tetID;          // ID of the belonging tetrahedron (-1 if boundary/missing side)
    float depth;        // Ray depth (t-value) of the intersection
};
```
The payload is stored in registers during traversal. In the raygen program, a static array or dynamic buffer is used to collect these payloads. Since the number of intersections varies per ray, the raygen program must track the count of collected hits.

**Numbered Steps:**
1.  **Ray Generation:** Initialize the payload array in the raygen program. Set `tmin` to a small epsilon (scene-size dependent) and `tmax` to a large value (e.g., $10^{20}$) to trace through the entire volume.
2.  **Trace:** Call `optixTrace` once with `OPTIX_RAY_FLAG_NONE` (enabling anyhit processing).
3.  **Anyhit Shader:** For every intersection:
    *   Compute the intersection attributes (barycentrics, tetID).
    *   Update the payload in registers using `optixSetPayload`.
    *   Continue traversal (do not abort) to find subsequent intersections.
4.  **Miss Shader:** If no hit is found, set the payload to a sentinel value (e.g., `depth = 10^{20}`) to signal termination in raygen.
5.  **Ray Generation (Collection):** After `optixTrace` returns, iterate through the collected payloads.
6.  **Sorting:** Sort the collected payloads in increasing order of `t` (depth). This is required because anyhit programs are not guaranteed to be called in t-order [S1, unofficial].
7.  **Processing:** Use the sorted intervals for volume rendering (e.g., integration).

**Formulas:**
*   **Ray Origin Update (for closehit alternative):** $O_{new} = O_{old} + D \cdot (t_{hit} + \epsilon)$
*   **Termination Condition:** If `payload.depth > Threshold` (e.g., $10^{15}$), stop processing.

**Parameters:**
*   `tmin`: Small epsilon (e.g., $10^{-6}$). Must be scene-size dependent [S1, unofficial].
*   `tmax`: Large value (e.g., $10^{20}$) [S1, unofficial].
*   `epsilon`: Offset added to hit position to avoid self-intersection.

**Pitfalls:**
*   **Ordering:** Anyhit calls are not ordered by t. Explicit sorting in raygen is mandatory [S1, unofficial].
*   **Performance:** Storing hits and sorting them consumes significant memory bandwidth [S1, unofficial].
*   **Epsilon Sensitivity:** If `tmin` epsilon is too small, self-intersections occur. If too large, adjacent tetrahedra at thin corners may be missed [S1, unofficial].
*   **Boundary Handling:** For boundary faces linking to only one tetrahedron, `tetID` may be -1, requiring special handling for barycentric coordinates [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Raygen Program
def raygen():
    # Initialize payload collection array
    hits = []
    
    # Define ray parameters
    tmin = 1e-6f  # Scene-size dependent epsilon
    tmax = 1e20f  # Trace through entire volume
    
    # Call trace with anyhit enabled (OPTIX_RAY_FLAG_NONE)
    optixTrace(
        traversable,
        origin,
        rayDir,
        tmin,
        tmax,
        0.0f,          # rayTime
        OptixVisibilityMask(255),
        OPTIX_RAY_FLAG_NONE, # Enables anyhit
        surfaceSBTIndex,
        rayTypeCount,
        missSBTIndex,
        payload_init
    )
    
    # Collect results from payload registers if using a single payload variable
    # Note: In practice, multiple hits require a buffer or atomic operations 
    # if using a single payload variable across multiple anyhit calls.
    # The OptiX 6 sample optixParticleVolumes uses a specific payload structure 
    # to accumulate hits. Here we assume a mechanism to collect them.
    
    # Sort hits by depth (t-value)
    hits.sort(key=lambda h: h.depth)
    
    # Process sorted hits for volume rendering
    for hit in hits:
        if hit.depth > 1e15f:
            break
        # Render interval between hit and next hit
        pass

# Anyhit Program
def anyhit():
    # Compute intersection details
    tetID = get_tet_id()
    barycentric = get_barycentrics()
    depth = get_intersection_t()
    
    # Update payload
    # In OptiX, this modifies the payload passed to the ray
    optixSetPayload_0(float_as_uint(depth))
    optixSetPayload_1(int_as_int(tetID))
    optixSetPayload_2_5(barycentric)
    
    # Continue traversal to find more hits

# Miss Program
def miss():
    # Signal no hit with a large depth value
    optixSetPayload_0(float_as_uint(1e20f))
```

## Subtopics

### Multi-Hit BVH Traversal
The parent technique of collecting N closest intersections. Multi-hit collection via anyhit is a specific implementation using OptiX's anyhit shader for tetrahedral meshes. [Multi-Hit BVH Traversal](12728-multi-hit-bvh-traversal.md)

### Closehit Iteration
An alternative approach calling `optixTrace` multiple times, advancing the ray origin after each hit. It avoids sorting but incurs per-ray-cast overhead. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Dominated by the sorting step in raygen, $O(K \log K)$ where $K$ is the number of intersections per ray. Traversal is $O(\log N)$ for BVH.
*   **Memory Complexity:** $O(K)$ to store hit payloads.
*   **Performance:** Avoids per-ray-cast overhead of multiple closehit calls but adds memory bandwidth costs for storing and sorting hits [S1, unofficial]. Performance depends on scene complexity and intersection count.
*   **Strengths:** Single ray cast, reduced API overhead.
*   **Limitations:** Memory bandwidth bottleneck, sorting overhead, sensitivity to epsilon values.
*   **When to Use:** When the number of intersections is moderate and the overhead of multiple ray casts is prohibitive.
*   **When Not to Use:** When memory bandwidth is a bottleneck or when hits can be processed out-of-order.

## Background
Originating from NVIDIA OptiX forums discussions on tetrahedral volume rendering, this technique was demonstrated in the OptiX 6 sample `optixParticleVolumes` and later in OptiX 7 samples `optixCutouts` and `optixWhitted` [S1, unofficial]. It addresses the limitation of closehit-only approaches in handling complex volumetric meshes with many intersections.

## Sources
[S1] How to collect all intersections using anyhit？ - OptiX - NVIDIA Developer Forums, https://forums.developer.nvidia.com/t/how-to-collect-all-intersections-using-anyhit/230282, fetched 2026-10-29 (unofficial)
