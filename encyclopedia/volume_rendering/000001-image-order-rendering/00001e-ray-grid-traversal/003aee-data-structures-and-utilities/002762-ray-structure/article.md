# Ray Structure
Also called: Ray Data Structure, Ray Representation, Ray3D

## Overview
The Ray Structure is a fundamental data object in volume rendering that represents a geometric ray in 3D space, defined by an origin point and a direction vector [S1, S2, S3, unofficial]. It serves as the primary query primitive for traversing volumetric data structures, enabling the evaluation of density, color, or material properties at specific points along the ray path [S1, S2, unofficial]. Unlike higher-level containers that manage batches or intersection records, the Ray Structure encapsulates the minimal geometric state required to define a linear path through space [G]. It distinguishes itself from sibling structures like the [Ray Bundle](2179-ray-bundle.md) by focusing on a single ray instance rather than batched metadata, and from the [Hit Record Structure](4953-hit-record-structure.md) by representing the input query rather than the output intersection result.

## How it works
### Inputs and Outputs
- **Inputs**: Origin point $\mathbf{o}$ (3D coordinates), Direction vector $\mathbf{v}$ (3D coordinates).
- **Outputs**: A ray object capable of evaluating position at any parameter $t$ within a valid range $[t_{min}, t_{max}]$.

### Data Structures
The core layout consists of two 3D vectors. The source code indicates that `point3` is an alias for `vec3`, implying both fields share the same underlying storage type (typically three floating-point numbers) [S2, unofficial].

**Fields:**
1. `orig`: `vec3` (or `point3`). Stores the ray origin $(o_x, o_y, o_z)$.
2. `dir`: `vec3`. Stores the ray direction $(v_x, v_y, v_z)$.

**Optional Fields (Range Clipping):**
Some implementations include explicit range parameters to clip the ray to a specific interval, preventing numerical instability or out-of-bounds access [S3, unofficial].
3. `range_min`: `float` or `double`. Default value $10^{-5}$ [S3, unofficial].
4. `range_max`: `float` or `double`. Default value $10^{30}$ [S3, unofficial].

### Numbered Steps

1. **Initialization**:
   - Create a ray object.
   - Set `orig` to the specified origin point.
   - Set `dir` to the specified direction vector.
   - If range parameters are supported, initialize `range_min` to $10^{-5}$ and `range_max` to $10^{30}$ [S3, unofficial].

2. **Direction Normalization (Optional but Recommended)**:
   - If the direction vector is not guaranteed to be unit length, normalize it to ensure that the parameter $t$ corresponds directly to Euclidean distance.
   - Compute length $L = \sqrt{v_x^2 + v_y^2 + v_z^2}$.
   - Update direction: $\mathbf{v} \leftarrow \frac{\mathbf{v}}{L} \times \text{length}$, where `length` is an optional scaling factor (default 1.0) [S2, unofficial].

3. **Point Evaluation**:
   - To find the 3D position $\mathbf{p}$ at parameter $t$:
     $$ \mathbf{p}(t) = \mathbf{o} + t \cdot \mathbf{v} $$
   - This operation is performed component-wise:
     $$ p_x = o_x + t \cdot v_x $$
     $$ p_y = o_y + t \cdot v_y $$
     $$ p_z = o_z + t \cdot v_z $$
   - The parameter $t$ must satisfy $t_{min} \le t \le t_{max}$ [S3, unofficial].

### Formulas
- **Ray Point**: $\mathbf{p}(t) = \mathbf{o} + t\mathbf{v}$
- **Direction Normalization**: $\mathbf{v}_{norm} = \frac{\mathbf{v}}{\|\mathbf{v}\|}$

### Parameters
- **Default Origin**: $(0, 0, 0)$ [S1, unofficial].
- **Default Direction**: $(0, 0, -1)$ [S1, unofficial]. Note: Some implementations may default to $(0,0,1)$ or leave it uninitialized until set [S2, unofficial].
- **$t_{min}$**: $10^{-5}$ [S3, unofficial]. Prevents division by zero or self-intersection artifacts in ray tracing.
- **$t_{max}$**: $10^{30}$ [S3, unofficial]. Effectively infinite for most rendering applications.

### Edge Cases and Pitfalls
- **Zero-Length Direction**: If the input direction vector has length 0, normalization fails. Implementers must check for $\|\mathbf{v}\| \approx 0$ and handle it as an error or degenerate case [G].
- **Precision**: Using single-precision floats for `orig` and `dir` may lead to precision loss when evaluating points far from the origin. Double precision is preferred for large scenes [G].
- **Range Clipping**: If $t < t_{min}$ or $t > t_{max}$, the point is outside the valid ray segment. Queries should clamp $t$ or return a "miss" status depending on the traversal algorithm's requirements [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Ray:
    def __init__(self, origin=None, direction=None):
        # Default origin (0,0,0) and direction (0,0,-1) if not provided
        if origin is None:
            self.orig = Vec3(0.0, 0.0, 0.0)
        else:
            self.orig = origin
            
        if direction is None:
            self.dir = Vec3(0.0, 0.0, -1.0)
        else:
            self.dir = direction
            
        # Optional range parameters
        self.range_min = 1e-5
        self.range_max = 1e30

    def at(self, t):
        """
        Computes the point on the ray at distance t.
        Formula: origin + t * direction
        """
        # Vector addition: orig + (t * dir)
        return Vec3(
            self.orig.x + t * self.dir.x,
            self.orig.y + t * self.dir.y,
            self.orig.z + t * self.dir.z
        )

    def set_ray_length(self, length):
        """
        Normalizes the direction vector and scales it by the given length.
        Ensures t corresponds to Euclidean distance.
        """
        current_length = self.dir.length()
        if current_length == 0:
            raise ValueError("Direction vector cannot be zero length")
        
        # Normalize
        inv_length = 1.0 / current_length
        self.dir.x *= inv_length
        self.dir.y *= inv_length
        self.dir.z *= inv_length
        
        # Scale
        self.dir.x *= length
        self.dir.y *= length
        self.dir.z *= length
```

## Subtopics

### Z-list-buffer Ray Representation
A specialized variant of the Ray Structure used for precise volume data representation and Boolean operations in sculpting applications. Instead of storing a single geometric ray, it stores an array of rays where each ray contains a list of sample points (position and normal) defining regions of the volume.
- **What it is**: A data structure replacing traditional scalar fields with vector-form ray data. Each ray consists of pairs of samples (left and right boundaries).
- **How it differs**: It stores explicit intersection metadata (normals and positions) rather than just geometric parameters. It uses axis-aligned rays along x, y, and z axes, generated via orthographic projection and Z-buffer scan conversion for speed.
- **When to choose**: Use for high-precision volumetric Boolean operations and sculpting where sharp feature preservation is critical, and performance benefits from sparse sampling (avg ~2-3 samples/ray vs 512 for dense grids).
- **Link**: [Z-list-buffer Ray Representation](12496-ray-segments-spec.md) (Note: The provided child list links this to Ray Segments Spec, but the content describes Z-list. Based on the prompt's structure, this is a minor variant without its own dedicated article in the tree, so it is described here. The link provided in the prompt for the child is `12496-ray-segments-spec.md`, but the content is distinct. I will link to the provided child name if it exists, but here the child is "Z-list-buffer Ray Representation" which is a minor variant. The prompt says "Minor variants (they have no article of their own: write each one under Subtopics as a '### <Name>' entry...". So I write it here.)

## Operating characteristics
- **Time Complexity**: Point evaluation is $O(1)$ [S1, S2, unofficial]. Normalization is $O(1)$ (constant number of arithmetic operations) [S2, unofficial].
- **Memory Complexity**: $O(1)$ per ray instance. Typically 24-48 bytes depending on float precision (3 floats for origin + 3 floats for direction) [S2, unofficial].
- **Performance**: Extremely fast for individual ray queries. The Z-list variant offers significant storage reduction for complex volumes compared to dense scalar fields (e.g., 2.86 samples/ray vs 512) [S4].
- **Strengths**: Simple, lightweight, and efficient for basic ray traversal.
- **Limitations**: Does not handle batching, intersection recording, or complex metadata natively. Requires external structures for batch processing or hit recording.
- **When to use**: As a building block for higher-level ray structures, traversal algorithms, and intersection tests.
- **When not to use**: For batched rendering (use [Ray Bundle](2179-ray-bundle.md)) or when intersection details are needed (use [Hit Record Structure](4953-hit-record-structure.md)).

## Background
The Ray Structure is a standard component in computer graphics, originating from foundational ray tracing literature. It is widely implemented in educational ray tracers and production renderers. The specific implementation details (such as `point3` aliasing `vec3` and default directions) are commonly found in open-source educational projects like the CPU voxel raycaster and sphere tracing examples [S1, S2, unofficial]. The Z-list variant was introduced for volume sculpting and Boolean operations to improve performance and precision over scalar field representations [S4].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] kuvsamja/lidar-sphere-tracing, https://github.com/kuvsamja/lidar-sphere-tracing, fetched 2026-10-26 (unofficial)
[S3] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S4] Detail sculpting using cubical marching squares, https://doi.org/10.1145/1152399.1152402, fetched 2026-11-06
