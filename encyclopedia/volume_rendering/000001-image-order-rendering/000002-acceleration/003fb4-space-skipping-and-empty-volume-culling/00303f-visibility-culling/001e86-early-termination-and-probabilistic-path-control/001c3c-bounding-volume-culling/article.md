# Bounding Volume Culling

Also called: Distance-based Culling, Ray Termination by Distance

## Overview
Bounding Volume Culling is an optimization that limits raymarching to a specific geometric boundary, dropping rays that exceed a certain distance from the eye or miss the volume entirely to speed up rendering [S1, unofficial][S3, unofficial][S4]. It differs from its parent, [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md), which relies on accumulated opacity or transmittance to stop integration, by using hard geometric constraints to discard rays before or during traversal [S1, unofficial][S2]. Unlike [Empty-space skipping](1002-empty-space-skipping.md), which skips empty voxels within a defined volume, Bounding Volume Culling often rejects the entire ray if it does not intersect the volume's outer bounds [S3, unofficial].

## How it works
The method varies by the type of bounding volume used (scalar distance limit, axis-aligned box, or heap-managed collection). The core principle is to compute a distance or intersection test against the bounding geometry before or during the sampling loop.

### Inputs and Outputs
*   **Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, bounding volume definition (e.g., max distance $D_{max}$, AABB min/max, or list of bounding volumes), and the volume data structure.
*   **Outputs:** A color value (or background) if the ray is culled or fully processed; otherwise, the result of the raymarching loop.

### Data Structures
1.  **Scalar Limit:** A single floating-point value $D_{max}$ representing the maximum travel distance [S1, unofficial].
2.  **AABB:** Two 3D vectors $\mathbf{min}$ and $\mathbf{max}$ defining the box extents [S3, unofficial][S4].
3.  **Heap (for Sphere Tracing):** A priority queue (min-heap) storing pairs of $(d_{bv}, \text{object\_id})$, where $d_{bv}$ is the distance to the bounding volume [S2].

### Algorithm: Scalar Distance Culling
This variant tracks the total distance traveled by the ray and terminates if it exceeds a predefined limit [S1, unofficial].

1.  Initialize accumulated distance $t = 0$.
2.  While $t < D_{max}$:
    a. Evaluate sample at $\mathbf{o} + t\mathbf{d}$.
    b. Update $t \leftarrow t + \Delta t$ (step size).
3.  If $t \ge D_{max}$, terminate and return background or fog color [S1, unofficial].

### Algorithm: AABB Culling
This variant uses the six planar faces of a rectangular grid volume to reject rays that do not intersect the volume [S3, unofficial][S4].

1.  Define the bounding box by $\mathbf{min}$ and $\mathbf{max}$.
2.  For each pixel, compute the intersection interval $[t_{enter}, t_{exit}]$ of the ray with the AABB.
    *   $t_{min} = \max(\frac{min_x - o_x}{d_x}, \frac{min_y - o_y}{d_y}, \frac{min_z - o_z}{d_z})$
    *   $t_{max} = \min(\frac{max_x - o_x}{d_x}, \frac{max_y - o_y}{d_y}, \frac{max_z - o_z}{d_z})$
    *   (Handle division by zero if $d_i = 0$ by treating the term as $-\infty$ or $+\infty$ appropriately).
3.  If $t_{max} < 0$ or $t_{min} > t_{max}$, the ray misses the volume.
4.  If the ray misses, assign a background/empty color and skip volume traversal [S3, unofficial][S4].
5.  If the ray hits, clamp the start of the march to $t_{min}$ (if $t_{min} > 0$) and end at $t_{max}$ [S4].

### Algorithm: Heap-based Bounding Volume Culling (Sphere Tracing)
This variant finds the closest object in a collection by maintaining a heap of bounding volume distances, culling based on the smallest bounding volume distance [S2].

1.  Compute distances $d_{bv}$ to each bounding volume in the collection.
2.  Create a min-heap of these distances.
3.  Initialize $d_{min} = \infty$.
4.  Loop:
    a. Let $d_{top}$ be the distance to the bounding volume at the top of the heap.
    b. Let $d_{content}$ be the distance to the contents of the object associated with $d_{top}$.
    c. Update $d_{min} = \min(d_{min}, d_{content})$.
    d. Remove the top of the heap.
    e. If the heap is empty, return $d_{min}$.
    f. Let $d_{next\_bv}$ be the distance to the bounding volume now at the top of the heap.
    g. If $d_{content} < d_{next\_bv}$, return $d_{min}$ (the closest object has been found).
    h. If $d_{min} < d_{next\_bv}$, return $d_{min}$ (a closer object was found earlier). [S2]

### Parameters
*   $D_{max}$: Maximum ray travel distance. Typical values depend on scene scale; e.g., 64 steps in some demonstrations [S1, unofficial].
*   $d_{bv}$: Distance to the bounding volume currently at the top of the heap [S2].

### Edge Cases and Pitfalls
*   **Perspective Degradation:** In perspective projection, bounding volume enhancements can degrade performance for typical implicit surfaces if the overhead of distance computation outweighs the culling benefit [S2].
*   **Precision:** Ray-AABB intersection requires careful handling of parallel rays (direction component zero) to avoid division by zero or incorrect interval calculations [S3, unofficial].
*   **Frustum Culling:** Bounding boxes can also be used to avoid iterations over objects outside the camera view frustum [S4].

## Subtopics

### [Bounding Spheres Optimization](1567-bounding-spheres-optimization.md)
Bounding Spheres Optimization reduces the computational cost of sphere tracing by grouping scene objects into larger enclosing spheres, thereby eliminating unnecessary distance calculations during ray traversal. It differs from its parent, [Bounding Volume Culling](1239-bounding-volume-culling.md), which typically discards rays entirely based on geometric bounds, by deferring the evaluation of individual object distances until the ray actually penetrates the enclosing volume. Unlike [Orthogonal Culling Optimization](4034-orthogonal-culling-optimization.md), which pre-computes object visibility via orthogonal projection to reduce per-step evaluations, Bounding Spheres Optimization dynamically manages the active set of distance fields based on ray proximity.

### Heap-based Bounding Volume Culling for Sphere Tracing
An algorithm to find the closest object in a collection by maintaining a heap of bounding volume distances. It computes distances to bounding volumes first, then contents, culling based on the smallest bounding volume distance [S2]. It is used when the scene consists of a union of implicit objects and sphere tracing is employed. It differs from simple scalar culling by managing a complex hierarchy of objects rather than a single distance limit [S2].

### Axis-Aligned Bounding Box (AABB) Culling
A spatial acceleration technique that uses the outer boundaries of the data volume to quickly reject rays that do not intersect the volume, avoiding unnecessary computation [S3, unofficial]. It is used for grid-based volumes where the extent is known. It differs from heap-based methods by using a single static box rather than a dynamic hierarchy of objects [S3, unofficial].

### Scene Delimitation with Bounding Boxes
Using bounding boxes to delimit cloud volumes and Smits' algorithm to limit ray-marching to the volume inside the cloud's boundaries, discarding rays that never hit the boundaries [S4]. It is used in real-time GPU volumetric cloud rendering. It differs from other methods by integrating with Smits' algorithm to calculate the specific segment of the Euclidean straight line for rendering and allowing separate voxel grids per cloud [S4].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for scalar and AABB culling per ray (intersection test). $O(N \log N)$ or $O(N)$ for heap-based culling depending on the number of bounding volumes $N$ and heap operations [S2].
*   **Memory Complexity:** $O(1)$ for scalar/AABB. $O(N)$ for heap-based culling to store the heap [S2].
*   **Performance:** AABB culling provides a rapid determination of a pixel's color when the data volume's projection does not affect it, reducing CPU/GPU time spent on ray location and object intersection [S3, unofficial]. Scene delimitation with bounding boxes increases FPS by discarding rays that never hit the boundaries and avoiding iterations over spheres outside the camera view frustum [S4].
*   **Limitations:** In perspective cases, bounding volume enhancements can degrade performance for typical implicit surfaces if the overhead is high [S2].
*   **When to use:** When the volume has a well-defined, limited extent (e.g., fog, clouds, specific objects) and rays frequently miss the volume or travel long distances through empty space [S1, unofficial][S3, unofficial][S4].

## Background
Bounding volume culling is a fundamental technique in computer graphics for accelerating ray tracing and ray marching. The use of bounding volumes to limit search space is one of the simplest techniques to reduce computation time for locating objects [S3, unofficial]. The concept of using bounding parallelepipeds for implicit surfaces was explored by Kay & Kajiya in 1986 [S2]. In the context of ray marching, limiting rays to a bounding volume is a common optimization to speed up rendering by dropping rays that exceed a certain distance [S1, unofficial]. Modern implementations often use GPU fragment shaders for bounding box delimitation with reduced linear cost [S4].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces, https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S3] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
[S4] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
