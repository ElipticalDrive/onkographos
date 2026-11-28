# Slice-by-Slice Ray Tracing

Also called: Data-driven ray tracing, Slab-based processing

## Overview
Slice-by-slice ray tracing is an object-order volume rendering algorithm that inverts the standard ray-tracing loop to be driven by data locality rather than ray traversal [S1]. It solves the memory bottleneck of random-access ray casting by loading only the necessary volume slices (slabs) into memory at any given time, making it suitable for large datasets that exceed available RAM [S1]. Unlike [Texture slicing](25-texture-slicing.md), which rasterizes proxy geometry to leverage hardware blending, this method performs explicit ray marching and compositing in software, maintaining precise control over the integration order and sample values [S1].

## How it works
**Inputs:** Volume dataset (scalar field), camera parameters (eye position, projection), ray spacing (resolution).
**Outputs:** 2D image (color and opacity per pixel).

**Data Structures:**
1.  **Ray State:** A structure storing per-ray state. Size: 36 bytes per ray [S1].
    *   `current_sample_point`: 3D coordinate of the current sample location (12 bytes, assumed float32) [S1].
    *   `difference_vector`: Step vector between consecutive sample points (12 bytes, assumed float32) [S1].
    *   `accumulated_color`: 3D color accumulator (12 bytes, assumed float32) [S1].
    *   `accumulated_opacity`: Scalar opacity accumulator (4 bytes, assumed float32) [S1].
    *   `current_sample_number`: Integer index of the current sample (4 bytes) [S1].
    *   `max_sample_number`: Integer index of the last sample within the volume (4 bytes) [S1].
    *   *Note:* The sources state the total size is 36 bytes. The fields listed above sum to 48 bytes if all are stored as 32-bit floats/integers. The text notes that space can be reduced by eliminating `current_sample_point` as it can be derived from the eye point, sample number, and difference vector [S1]. For the standard 36-byte implementation described, the exact bit-packing of the 36 bytes is not explicitly detailed beyond the field list, but the fields are: current sample point, difference vector, color, opacity, current sample number, and maximum sample number [S1].
2.  **Slab Queue:** A linked list of pointers to Ray State structures. Maintained as a queue where objects are added and removed from the front [S1]. There is one queue per slab.

**Algorithm Steps:**

1.  **Initialization:**
    For each pixel (ray) in the image:
    a. Trace the ray from the eye point to the first sample point inside the volume grid.
    b. Determine the slab index $k$ that contains this first sample point. A slab is defined as a pair of adjacent slices required for trilinear interpolation [S1].
    c. Initialize the Ray State with the first sample point, the difference vector (step size), initial color/opacity (0), and sample counters.
    d. Place a pointer to this Ray State onto the slab queue for slab $k$.

2.  **Processing Loop:**
    While any slab queue is non-empty:
    a. Select a slab $k$ to load into memory. The processing order should be front-to-back (or back-to-front) to ensure correct compositing without reloading slabs [S1].
    b. Load the two slices defining slab $k$ into memory.
    c. For each ray pointer $R$ on the queue for slab $k$:
        i.  Let $P$ be the `current_sample_point` of $R$.
        ii. Compute the sample value $V$ at $P$ using trilinear interpolation from the loaded slab [S1].
        iii. Compute the accumulated color $C$ and opacity $A$ using the volume rendering integral step (e.g., front-to-back compositing):
            $$ C_{new} = C_{old} + V \cdot A_{old} \cdot (1 - A_{old}) $$
            $$ A_{new} = A_{old} + V \cdot (1 - A_{old}) $$
            *(Note: The specific compositing equation is not explicitly written in the source text, but the fields `color` and `opacity` are maintained [S1]. Standard volume rendering compositing is implied by the context of "final pixel value" and "opacity")*.
        iv. Compute the location of the next sample point: $P_{next} = P + \text{difference\_vector}$.
        v.  Check if $P_{next}$ is still inside the current slab $k$.
            *   If yes, update $R$'s `current_sample_point` to $P_{next}$, update color/opacity, and continue to the next iteration of the inner loop.
            *   If no, check if $P_{next}$ is still inside the volume boundary.
                *   If yes, determine the new slab index $j$ for $P_{next}$ and place $R$ onto the slab queue for slab $j$.
                *   If no, the ray has exited the volume; terminate processing for this ray.

**Parameters:**
*   **Memory Storage:** The method requires $36m^2 + 8n^2$ bytes, where $m$ is the screen resolution (number of rays) and $n$ is the data resolution (side length of the volume grid) [S1].
*   **Comparison:** Storing the entire dataset requires $8n^3$ bytes [S1]. The method is more memory efficient than storing the full volume when $n > 6$ (assuming screen resolution equals data resolution, i.e., $m=n$) [S1].

**Edge Cases and Pitfalls:**
*   **Slab Order:** Perspective rays diverge. A processing order must be chosen such that each slab is loaded only once. Front-to-back processing requires loading slabs in increasing depth order; back-to-front requires decreasing depth order [S1].
*   **Deriving Position:** To save memory, the `current_sample_point` can be omitted from the Ray State and computed from the eye point, sample number, and difference vector, reducing the per-ray storage [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the average number of samples per ray. The overhead of queue management is negligible (two pointer reassignments per enqueue/dequeue) [S1].
*   **Memory Complexity:** $O(m^2 + n^2)$, where $m$ is screen resolution and $n$ is volume resolution. This is significantly lower than the $O(n^3)$ required for full volume storage [S1].
*   **Performance:** Efficient for large volumes where $n$ is large, as it avoids loading the entire dataset into RAM. Performance is limited by memory bandwidth for loading slabs and CPU cache efficiency for ray state access [S1].
*   **Strengths:** Memory efficiency for large datasets; precise control over ray marching and compositing; no need for complex geometry generation or rasterization overhead.
*   **Limitations:** Requires explicit ray traversal logic; less hardware-accelerated than texture slicing methods; cache misses may occur if ray states are scattered in memory.

## Background
The method was introduced by Levoy in 1990 in "An efficient method for volume rendering using perspective projection" [S1]. It was developed to address the memory limitations of early volume rendering systems, which required storing the entire volume dataset in RAM for random access. By inverting the ray-tracing loop to be data-driven, the algorithm enabled the rendering of large datasets on hardware with limited memory.

## Sources
[S1] An efficient method for volume rendering using perspective projection, https://doi.org/10.1145/99308.99329, fetched 2026-10-26
