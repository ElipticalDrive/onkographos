# Ray Structure

Also called: Ray Data Structure, Ray Representation, Ray3D

## Overview
A Ray Structure is a geometric primitive class representing a ray in 3D space, defined by an origin point and a direction vector [S1, S2, S3, S4, unofficial]. It serves as the fundamental unit for image-order volume rendering traversal, providing methods to evaluate points along the ray path [S1, S2, S3, S4, unofficial]. It differs from sibling structures like the [Ray Bundle](2179-ray-bundle.md) by managing a single ray's state rather than batched metadata, and from the [Hit Record Structure](4953-hit-record-structure.md) by representing the query vector rather than the intersection result [G].

## How it works
**Inputs and Outputs:**
The structure takes an origin $\mathbf{o}$ and a direction $\mathbf{v}$ as inputs. It outputs a point $\mathbf{p}$ on the ray for a given scalar parameter $t$, or a normalized direction vector.

**Data Structures and Layout:**
The core layout consists of two 3D vectors:
1.  `orig`: The ray origin, typically a `point3` or `vec3` type (3 floating-point values) [S2, S4, unofficial].
2.  `dir`: The ray direction, a `vec3` type (3 floating-point values) [S2, S4, unofficial].

Some implementations extend this with additional metadata for traversal context:
*   `depth`: An unsigned 32-bit integer (`u32`) representing traversal depth [S4, unofficial].
*   `time`: A 32-bit float (`f32`) for time-varying scenes [S4, unofficial].
*   `range_min` / `range_max`: Floating-point bounds for the valid parameter $t$ [S3, unofficial].

**Numbered Steps:**
1.  **Initialization:**
    *   Default constructor: Initialize `orig` to $(0,0,0)$ and `dir` to $(0,0,-1)$ [S1, S3, unofficial].
    *   Parameterized constructor: Accept `orig` and `dir`. Normalize `dir` to unit length. Set `range_min` to $10^{-5}$ and `range_max` to $10^{30}$ [S3, unofficial].
    *   Length setting: If a specific ray length $L$ is required, normalize `dir` and scale it: $\mathbf{v}_{new} = \frac{\mathbf{v}}{\|\mathbf{v}\|} \times L$ [S2, S3, unofficial].
2.  **Point Evaluation:**
    *   Compute the point at parameter $t$ using the linear equation:
        $$ \mathbf{p}(t) = \mathbf{o} + t \cdot \mathbf{v} $$
    *   This operation assumes $\mathbf{v}$ is normalized if $t$ represents Euclidean distance [S1, S2, S3, S4, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Ray:
    def __init__(self, origin=None, direction=None):
        if origin is None:
            self.orig = [0.0, 0.0, 0.0]
        else:
            self.orig = list(origin)
            
        if direction is None:
            self.dir = [0.0, 0.0, -1.0]
        else:
            self.dir = list(direction)
            self._normalize_direction()
            
        # Default range bounds [S3, unofficial]
        self.range_min = 1e-5
        self.range_max = 1e30

    def _normalize_direction(self):
        """Normalize direction vector to unit length."""
        x, y, z = self.dir
        length = (x**2 + y**2 + z**2)**0.5
        if length > 0:
            self.dir[0] /= length
            self.dir[1] /= length
            self.dir[2] /= length

    def set_ray_length(self, length):
        """Scale direction vector to a specific Euclidean length."""
        x, y, z = self.dir
        current_length = (x**2 + y**2 + z**2)**0.5
        if current_length > 0:
            scale = length / current_length
            self.dir[0] *= scale
            self.dir[1] *= scale
            self.dir[2] *= scale

    def at(self, t):
        """
        Compute point on ray at parameter t.
        Formula: p = orig + t * dir
        """
        return [
            self.orig[0] + t * self.dir[0],
            self.orig[1] + t * self.dir[1],
            self.orig[2] + t * self.dir[2]
        ]

    def get_point(self, t):
        """Alias for at(t)."""
        return self.at(t)
```

**Parameters:**
*   `RAY_MIN_T`: Minimum valid $t$ value, typically $10^{-5}$ to avoid self-intersection or precision errors [S3, unofficial].
*   `RAY_MAX_T`: Maximum valid $t$ value, typically $10^{30}$ or scene bounds [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Direction:** If the input direction is $(0,0,0)$, normalization fails. Implementations should guard against this or treat it as a degenerate ray [G].
*   **Precision:** Using unnormalized directions with $t$ as distance leads to incorrect sampling positions. Always normalize or use $t$ as a scalar multiplier of the raw vector [S2, S3, unofficial].
*   **Range Clipping:** In traversal, $t$ must be clamped to $[range\_min, range\_max]$ to prevent sampling outside the volume or at the camera plane [S3, unofficial].

## Subtopics

### [Ray Bundle](2179-ray-bundle.md)
A Ray Bundle is a container data structure that groups ray origins, directions, and associated sampling parameters (sample points and lengths) into a single object, typically implemented as a collection of aligned tensors. It solves the problem of managing batched ray data for volume rendering and neural rendering pipelines, allowing efficient vectorized operations and memory transfers. It differs from a [RayBatch container](4187-raybatch-container.md) by explicitly bundling sampling metadata (sample points/lengths) alongside geometric ray data, and from a [Hit Record Structure](4953-hit-record-structure.md) by focusing on input ray generation rather than output intersection results.

### [RayBatch container](4187-raybatch-container.md)
The RayBatch container is a NumPy-style data structure for storing and sampling ray batches during training, supporting efficient indexing, slicing, and conversion to Mitsuba tensor types. It provides batch-level operations and explicit conversion logic to Mitsuba 3, distinguishing it from the generic [Ray Structure](4026-ray-structure.md) and the [Ray Bundle](2179-ray-bundle.md) [G].

### [Hit Record Structure](4953-hit-record-structure.md)
A hit record structure stores the geometric and material properties of a ray-surface intersection. It passes intersection context (position, normal, distance, and material reference) from traversal to shading. It is distinguished from sibling structures like the [Ray Bundle](2179-ray-bundle.md) or [RayBatch container](4187-raybatch-container.md) by focusing on the state of a single intersection event rather than the propagation or batching of rays themselves [G].

### [Plücker Ray Embedding](5132-pl-cker-ray-embedding.md)
Plücker Ray Embedding parameterizes a 3D ray using Plücker coordinates to create a scale-, position-, and focal-length-independent representation. It maps a ray defined by origin $\mathbf{o}$ and direction $\mathbf{v}$ into a 16-dimensional multivector, enabling robust modulation of camera poses to image features in neural rendering frameworks. Unlike extrinsic/intrinsic matrix parameterizations, this method relies on the cross product $\mathbf{o} \times \mathbf{v}$.

### [Z-list-buffer Ray Representation](11281-z-list-buffer-ray-representation.md)
The Z-list-buffer Ray Representation is a volume data storage format that stores voxel data as an array of rays, where each ray contains a list of sample points (position and normal) defining regions. It replaces traditional scalar fields to reduce storage and preserve sharp features, supporting any cell-based triangulation method. It is distinguished from sibling structures by its specific generation method using orthographic projection and a Z-buffer with a Z-list, which is faster than modified ray tracing for axis-aligned rays.

### [Ray Segments Spec](12496-ray-segments-spec.md)
Ray Segments Spec is a data structure designed to represent ray segments efficiently, particularly in the context of volume rendering and neural radiance fields. It stores segment values, endpoint flags, validity, chunking information, and ray mappings. It differs from sibling structures like [Ray Bundle](2179-ray-bundle.md) by focusing on explicit segment-level metadata rather than raw ray parameters, and from [RayBatch container](4187-raybatch-container.md) by providing granular control over segment validity and chunking.

### Ray and Bounding Box Structures
**Definition:** Core geometric primitives used for ray tracing and scene representation: Ray (origin, direction, depth, time), DifferentialRay (extends Ray with perturbation origins/directions), and BBox (min/max points) [S4, unofficial].

**Layout:**
*   **Ray:**
    *   `ori` (Point3): 12 bytes [S4, unofficial].
    *   `dir` (Vec3): 24 bytes [S4, unofficial].
    *   `depth` (u32): 4 bytes [S4, unofficial].
    *   `time` (f32): 4 bytes [S4, unofficial].
    *   Total: 44 bytes [S4, unofficial].
*   **BBox:**
    *   `minP`, `maxP` (Point3).
    *   Initialized to `{INFINITY, INFINITY, INFINITY}` and `{-INFINITY, -INFINITY, -INFINITY}` respectively [S4, unofficial].

**Code:** Point evaluation follows the standard linear interpolation $\mathbf{p} = \mathbf{o} + t\mathbf{v}$ [S4, unofficial].*(Note: No dedicated article exists for this variant, so it is listed here as a minor variant without a specific link, or linked to the closest sibling if applicable. As per instructions, minor variants have no article of their own.)*

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** $O(1)$ per ray. Typically 24 bytes (origin + direction) or 44 bytes with metadata [S2, S4, unofficial].
*   **Time:** Point evaluation is $O(1)$, involving 3 multiplications and 3 additions [S1, S2, S3, S4, unofficial].

**Measured Performance:**
Performance is dominated by the cost of the evaluation function when called millions of times during traversal. Vectorization (SIMD) benefits are realized when using [Ray Bundle](2179-ray-bundle.md) or [RayBatch container](4187-raybatch-container.md) structures that align multiple rays in memory [S1, S2, S3, S4, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Minimal memory footprint, simple arithmetic, hardware-friendly layout [S1, S2, S3, S4, unofficial].
*   **Limitations:** Does not handle batch operations natively (requires external containers), lacks metadata for complex traversal states (e.g., hit records) [G].

**When to use:**
Use for single-ray queries, basic traversal kernels, or as a building block for batched structures. Avoid for neural rendering pipelines requiring batched tensor operations; use [Ray Bundle](2179-ray-bundle.md) instead.

## Background
The Ray Structure is a fundamental concept in computer graphics, originating from standard ray tracing literature. It is widely implemented in open-source volume rendering projects such as `cpu_voxel_raycaster` [S1, unofficial], `lidar-sphere-tracing` [S2, unofficial], and `volume_renderer` [S3, unofficial]. Variants with extended metadata (depth, time) are found in real-time rendering demos like `SVODemo` [S4, unofficial].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] kuvsamja/lidar-sphere-tracing, https://github.com/kuvsamja/lidar-sphere-tracing, fetched 2026-10-26 (unofficial)
[S3] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S4] Throbbing/SVODemo, https://github.com/Throbbing/SVODemo, fetched 2026-10-26 (unofficial)
