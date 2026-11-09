# Ray Structure

Also called: Ray Data Structure, Ray Representation, Ray3D

## Overview
The Ray Structure is a class representing a geometric ray in 3D space, defined by an origin point and a direction vector [S1, unofficial]. It provides methods to evaluate points along the ray parameter and manage ray range bounds [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Origin point $\mathbf{o} \in \mathbb{R}^3$, Direction vector $\mathbf{d} \in \mathbb{R}^3$.
*   **Output:** A ray object capable of evaluating points at parameter $t$ and providing access to origin and direction.

**Data Structures:**
The ray is stored as a class or struct with two primary fields [S2, S3, unofficial]:
1.  `orig`: The origin point, typically a 3D vector (`point3` or `vec3`) [S2, unofficial].
2.  `dir`: The direction vector, typically a 3D vector (`vec3`) [S2, unofficial].
`point3` is an alias for `vec3` [S2, unofficial].

**Parameters:**
*   `RAY_MIN_T`: The minimum valid parameter $t$ for the ray. Default value is $10^{-5}$ [S3, unofficial].
*   `RAY_MAX_T`: The maximum valid parameter $t$ for the ray. Default value is $10^{30}$ [S3, unofficial].

**Steps:**
1.  **Initialization:**
    *   The default constructor initializes `orig` to $(0, 0, 0)$ and `dir` to $(0, 0, -1)$ [S1, unofficial].
    *   Alternatively, the default constructor may initialize `orig` and `dir` to zero vectors, with `range_min` set to `RAY_MIN_T` and `range_max` set to `RAY_MAX_T` [S3, unofficial].
2.  **Construction:**
    *   A parameterized constructor accepts an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [S2, S3, unofficial].
    *   The direction vector is normalized to unit length and scaled by a specified length (if provided) [S2, S3, unofficial]. Specifically: $\mathbf{d}_{normalized} = \frac{\mathbf{d}}{\|\mathbf{d}\|} \times \text{length}$ [S2, unofficial].
    *   The range parameters `range_min` and `range_max` are set to their default values ($10^{-5}$ and $10^{30}$) unless explicitly overridden [S3, unofficial].
3.  **Point Evaluation:**
    *   The method `at(t)` computes the point on the ray at distance $t$ using the parametric equation:
        $$ P(t) = \mathbf{o} + t \cdot \mathbf{d} $$
        where $\mathbf{o}$ is the origin, $\mathbf{d}$ is the direction, and $t$ is the scalar parameter [S1, S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Ray:
    def __init__(self, origin=None, direction=None, length=1.0):
        # Default initialization if no arguments provided
        if origin is None:
            self.orig = [0.0, 0.0, 0.0]
        else:
            self.orig = list(origin)
            
        if direction is None:
            self.dir = [0.0, 0.0, -1.0]  # S1 default direction
        else:
            self.dir = list(direction)
            
        # Set ray range parameters
        self.range_min = 1e-5  # S3
        self.range_max = 1e30  # S3
        
        # Normalize direction and apply length
        self._normalize_and_scale(length)
        
    def _normalize_and_scale(self, length):
        # Calculate magnitude
        mag = (self.dir[0]**2 + self.dir[1]**2 + self.dir[2]**2)**0.5
        if mag > 1e-9:  # Guard against zero direction [G]
            # Normalize
            self.dir[0] /= mag
            self.dir[1] /= mag
            self.dir[2] /= mag
            # Scale by length
            self.dir[0] *= length
            self.dir[1] *= length
            self.dir[2] *= length

    def at(self, t):
        # Compute point on ray: origin + t * direction
        return [
            self.orig[0] + t * self.dir[0],
            self.orig[1] + t * self.dir[1],
            self.orig[2] + t * self.dir[2]
        ]

    def set_ray_length(self, length):
        # Re-normalize direction and apply new length
        mag = (self.dir[0]**2 + self.dir[1]**2 + self.dir[2]**2)**0.5
        if mag > 1e-9:
            self.dir[0] /= mag
            self.dir[1] /= mag
            self.dir[2] /= mag
            self.dir[0] *= length
            self.dir[1] *= length
            self.dir[2] *= length
```

**Edge Cases and Pitfalls:**
*   **Zero Direction:** If the input direction vector is zero, normalization fails. The implementation must handle this, either by keeping the zero vector or raising an error [G].
*   **Range Clipping:** The `range_min` and `range_max` parameters define the valid interval for $t$ during traversal [S3, unofficial].

## Subtopics

### [Ray Bundle](2179-ray-bundle.md)
A Ray Bundle is a container data structure that groups ray origins, directions, and associated sampling parameters (sample points and lengths) into a single object, typically implemented as a collection of aligned tensors [G]. It solves the problem of managing batched ray data for volume rendering and neural rendering pipelines, allowing efficient vectorized operations and memory transfers [G]. It differs from a [RayBatch container](4187-raybatch-container.md) by explicitly bundling sampling metadata (sample points/lengths) alongside geometric ray data, and from a [Hit Record Structure](4953-hit-record-structure.md) by focusing on input ray generation rather than output intersection results [G].

### [RayBatch container](4187-raybatch-container.md)
The RayBatch container is a NumPy-style data structure for storing and sampling ray batches during training, supporting efficient indexing, slicing, and conversion to Mitsuba tensor types. It provides batch-level operations and explicit conversion logic to Mitsuba 3, distinguishing it from the generic [Ray Structure](4026-ray-structure.md) and the [Ray Bundle](2179-ray-bundle.md) [G].

### [Hit Record Structure](4953-hit-record-structure.md)
The Hit Record Structure is a data container used in ray tracing and volume rendering to store the geometric and material properties of a ray-surface intersection [G]. It solves the problem of passing intersection context (position, normal, distance, and material reference) from the traversal phase to the shading phase [G]. It is distinguished from sibling structures like the [Ray Bundle](2179-ray-bundle.md) or [RayBatch container](4187-raybatch-container.md) by focusing on the state of a single intersection event rather than the propagation or batching of rays themselves [G].

#### Plücker Ray Embedding
Plücker Ray Embedding embeds a ray defined by origin $\mathbf{o}$ and direction $\mathbf{v}$ into a 16-dimensional multivector using Plücker coordinates $(\mathbf{v}, \mathbf{o} \times \mathbf{v})$ [S4, unofficial]. The input tensor has shape `[..., 6]` and the output has shape `[..., 16]` [S4, unofficial]. The process involves initializing a zero tensor `mv` of shape `[..., 16]` with the same device and dtype as the input `pluecker_ray` [S4, unofficial]. The direction vector $\mathbf{v}$ (indices 3:6 of input) is assigned to multivector indices 5:8 [S4, unofficial]. The cross product component $\mathbf{o} \times \mathbf{v}$ is assigned to multivector indices 8, 9, 10, where index 8 receives input index 2, index 9 receives negative input index 1, and index 10 receives input index 0 [S4, unofficial]. The Plücker ray is extracted from the multivector by concatenating: multivector index 10, negative multivector index 9, multivector index 8, and multivector indices 5:8 [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** $O(1)$ per ray, storing 6 floating-point values (3 for origin, 3 for direction) [S2, S3, unofficial].
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The Ray Structure originates from the fundamental need to represent rays in 3D space for rendering and intersection testing [S1, S2, S3, unofficial]. It serves as a basic building block in computer graphics implementations [S1, S2, S3, unofficial].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] kuvsamja/lidar-sphere-tracing, https://github.com/kuvsamja/lidar-sphere-tracing, fetched 2026-10-26 (unofficial)
[S3] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S4] NorahCadeJS/GAI-GS, https://github.com/NorahCadeJS/GAI-GS, fetched 2026-10-27 (unofficial)
