# Splat Raycasting Dispatch


## Overview
Splat Raycasting Dispatch is the computational kernel that performs ray-splat intersection tests for 3D Gaussian splats, returning hit distances and splat indices. It solves the problem of determining which splats intersect a given ray within a specified depth range, serving as the core intersection primitive for raycasting-based Gaussian splatting renderers. It differs from [Ray-Splat Intersection](6816-ray-splat-intersection.md) by operating on pre-transformed, packed buffer structures optimized for batched processing, and from [Spherical Panoramic Gaussian Splatting (SPaGS)](3042-spherical-panoramic-gaussian-splatting-spags.md) by targeting standard Euclidean camera models rather than spherical panoramas.

## How it works
The method processes a batch of splats against a single ray (or a set of rays if extended, though the core dispatch is per-ray or per-tile in the implementation context). It relies on a shared memory layout and a specific buffer management strategy to minimize overhead.

### Inputs and Outputs
**Inputs:**
- `splats`: A flat array of `u32` values representing splat data. Each splat is represented by 4 `u32` words (16 bytes). The layout of these words is defined by the parent [Ray-Splat/Gaussian Intersection](7916-ray-splat-gaussian-intersection.md) article.
- `splats2`: A secondary buffer of `u32` values, size `4 * count`, used for auxiliary data or intermediate calculations during intersection.
- `origin`: Ray origin coordinates $(o_x, o_y, o_z)$ as `f32`.
- `direction`: Ray direction vector $(d_x, d_y, d_z)$ as `f32`.
- `min_opacity`: Minimum opacity threshold `f32` for considering a splat as a hit.
- `near`: Near plane distance `f32`.
- `far`: Far plane distance `f32`.
- `tables`: Precomputed lookup tables `RaycastTables` used for optimization (e.g., bounding volume checks or opacity thresholds).

**Outputs:**
- `distances`: A `Vec<f32>` of size `RAYCAST_BUFFER_COUNT` containing the intersection distances for each splat.
- `indices`: A `Vec<u32>` of capacity `RAYCAST_BUFFER_COUNT` containing the original splat indices that resulted in a hit.

### Data Structures
The implementation uses a fixed-capacity buffer structure to manage memory efficiently.

**Constants:**
- `RAYCAST_BUFFER_COUNT`: `usize`, set to 65536. This defines the maximum number of splats that can be processed in a single dispatch batch.

**Structures:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
struct RaycastBuffers {
    splats: Vec<u32>,       # Size: RAYCAST_BUFFER_COUNT * 4
    splats2: Vec<u32>,      # Size: RAYCAST_BUFFER_COUNT * 4
    distances: Vec<f32>,    # Size: RAYCAST_BUFFER_COUNT
    indices: Vec<u32>,      # Capacity: RAYCAST_BUFFER_COUNT
    tables: RaycastTables   # Precomputed optimization tables
}
```

### Algorithm Steps
1. **Buffer Initialization**: Clear the `distances` and `indices` buffers. Set all `distances` to infinity or a value indicating no hit.
2. **Subbuffer Extraction**: Extract the relevant portion of the input splat arrays. If `count` splats are to be processed, extract `splats[0..(4 * count)]` and `splats2[0..(4 * count)]`.
3. **Validation**: Ensure `count` as `usize` does not exceed `RAYCAST_BUFFER_COUNT`. If it does, throw an error [S1, unofficial].
4. **Dispatch Call**: Invoke the core intersection function `raycast_splat_ellipsoids` with the extracted subbuffers, the output buffers, ray parameters, and tables.
   - Function signature: `raycast_splat_ellipsoids(subbuffer, subbuffer2, distances, indices, origin, direction, min_opacity, tables, near, far)`
5. **Result Retrieval**: Return the `distances` buffer as a `Float32Array` view. The `indices` buffer contains the valid splat indices corresponding to non-infinite distances.

### Pseudocode
The following pseudocode represents the dispatch logic in a Python-like style, assuming the existence of the underlying WASM or native `raycast_splat_ellipsoids` function.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splat_ray_casting_dispatch(splats, splats2, count, origin, direction, min_opacity, near, far, tables):
    """
    Dispatches ray-splat intersection tests for a batch of splats.
    
    Args:
        splats (list): Flat list of u32 splat data, size >= 4 * count.
        splats2 (list): Flat list of u32 auxiliary data, size >= 4 * count.
        count (int): Number of splats to process.
        origin (tuple): (ox, oy, oz) Ray origin.
        direction (tuple): (dx, dy, dz) Ray direction.
        min_opacity (float): Minimum opacity threshold.
        near (float): Near plane distance.
        far (float): Far plane distance.
        tables (object): Precomputed raycast tables.
        
    Returns:
        tuple: (distances, indices) where distances is a list of f32 and indices is a list of u32.
    """
    RAYCAST_BUFFER_COUNT = 65536
    
    # Validate count
    if count > RAYCAST_BUFFER_COUNT:
        raise ValueError(f"Count {count} exceeds buffer capacity {RAYCAST_BUFFER_COUNT}")
    
    # Initialize output buffers
    distances = [float('inf')] * RAYCAST_BUFFER_COUNT
    indices = [0] * RAYCAST_BUFFER_COUNT
    
    # Extract subbuffers
    # Note: In a real implementation, this might be a view or slice to avoid copying
    subbuffer_splats = splats[0 : 4 * count]
    subbuffer_splats2 = splats2[0 : 4 * count]
    
    # Call the core intersection kernel
    # This function is assumed to be implemented in Rust/WASM and exposed via bindings
    raycast_splat_ellipsoids(
        subbuffer_splats,
        subbuffer_splats2,
        distances,
        indices,
        origin[0], origin[1], origin[2],
        direction[0], direction[1], direction[2],
        min_opacity,
        tables,
        near,
        far
    )
    
    # Return the results
    # The indices buffer may contain zeros for non-hits; filtering is typically done by the caller
    return distances, indices
```

### Edge Cases and Pitfalls
- **Buffer Overflow**: The `count` parameter must not exceed `RAYCAST_BUFFER_COUNT` (65536). Exceeding this limit causes an error [S1, unofficial].
- **Opacity Threshold**: `min_opacity` must be strictly less than 1.0. If it is 1.0 or greater, no splats will be considered hits, as opacity is typically normalized to [0, 1] [S1, unofficial].
- **Precision**: The use of `f32` for distances and coordinates may lead to precision issues at very large scene scales or very small splat sizes. The parent article [Ray-Splat/Gaussian Intersection](7916-ray-splat-gaussian-intersection.md) discusses coordinate space transformations to mitigate this.

## Subtopics

### Splat Raycasting Implementation
This variant describes the higher-level integration of the dispatch kernel within a JavaScript/WASM environment, specifically for the Gaussian-Splat-Lite project. It handles coordinate transformation, buffer management, and result post-processing.
- **Process**:
  1. Transform ray origin and direction from world space to mesh space using the inverse of `matrixWorld`. Store in `raycastOrigin` and `raycastDirection` [S1, unofficial].
  2. Iterate over splat ranges using `splats.forEachRaycastRange`. For each range, copy splat records into the dispatch buffers using `splats.copySplatRecords`, mapping original indices to `raycastVisibleIndices`. Flush to WASM when buffer capacity is reached [S1, unofficial].
  3. On flush, call `raycast_splat_buffers` with the transformed origin, direction, `minRaycastOpacity` (default 0.1), `near`, `far`, and `count`. Retrieve `distances` and `hitIndices` from WASM [S1, unofficial].
  4. For each hit, compute the world point via `ray.at(distance)` and store the distance, point, object, and original splat index in the `intersects` list [S1, unofficial].
- **Data Structures**:
  - `raycastVisibleIndices`: A `Uint32Array` mapping buffer slot indices to original splat indices, sized to `buffer.length / 4`. It is reallocated if capacity changes [S1, unofficial].
- **Parameters**:
  - `minRaycastOpacity`: Threshold for raycast inclusion, defaults to 0.1. Must be strictly less than 1.0 [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ per ray, where $N$ is the number of splats in the batch. The complexity is linear because each splat is tested independently against the ray.
- **Memory Complexity**: $O(N)$ for the input and output buffers, where $N$ is the batch size. The fixed buffer size of 65536 splats means memory usage is bounded but may be underutilized for small scenes.
- **Performance**: The method is optimized for batched processing on WASM, leveraging SIMD instructions via `RUSTFLAGS="-C target-feature=+simd128,+bulk-memory"` [S1, unofficial]. Performance scales with the number of splats per batch, up to the buffer limit.
- **Strengths**: High throughput for large numbers of splats due to batched processing and SIMD optimization. Low overhead for individual ray tests.
- **Limitations**: Fixed buffer size limits the maximum batch size. Requires careful management of buffer flushing and coordinate transformations in the host language.

## Background
Splat Raycasting Dispatch is part of the Gaussian-Splat-Lite project, an implementation of 3D Gaussian Splatting for web browsers using WebAssembly [S1, unofficial]. It was developed to enable real-time rendering of Gaussian splats in the browser by offloading the computationally intensive ray-splat intersection tests to a WASM module. The method builds on the geometric principles of ray-splat intersection defined in the parent article [Ray-Splat/Gaussian Intersection](7916-ray-splat-gaussian-intersection.md).

## Sources
[S1] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
