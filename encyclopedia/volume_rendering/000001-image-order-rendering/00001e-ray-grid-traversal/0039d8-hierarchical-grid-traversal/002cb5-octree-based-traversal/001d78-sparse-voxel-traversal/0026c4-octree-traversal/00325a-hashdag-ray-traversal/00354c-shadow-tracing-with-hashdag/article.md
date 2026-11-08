# Shadow Tracing with HashDAG


## Overview
This method computes the final pixel color for a ray-grid traversal by determining if the ray's path through the volume is occluded by the scene geometry (shadowed) and applying lighting and fog effects. It differs from standard volume rendering by treating the HashDAG as an implicit geometry for shadow testing rather than integrating density along the ray, relying on the parent [HashDAG Ray Traversal](6745-hashdag-ray-traversal.md) to provide the ray path.

## How it works
**Inputs:**
- `params`: A structure containing:
  - `colorsSurface`: Output buffer for pixel colors.
  - `pathsSurface`: Buffer containing ray path data (origin, etc.).
  - `rayMin`, `rayDDx`, `rayDDy`: Camera ray parameters.
  - `cameraPosition`: Camera origin in world space.
  - `shadowBias`: Scalar bias for shadow ray origin offset.
  - `fogDensity`: Base fog density scalar.
- `dag`: The HashDAG structure representing the volume geometry.
- `sun_direction`: Normalized vector for the light source.

**Outputs:**
- Updated `params.colorsSurface` with the computed `uint32` RGB888 color for the current pixel.

**Data Structures:**
- `Path`: Loaded from `params.pathsSurface`. Contains `rayOrigin` (float3).
- `float3`: 3-component floating-point vector.
- `double3`: 3-component double-precision floating-point vector.

**Parameters:**
- `EXACT_SHADOWS`: Boolean flag. If true, shadow start is the intersection point; otherwise, it is the ray origin.
- `PER_VOXEL_FACE_SHADING`: Boolean flag. If true, uses voxel face normals for lighting; otherwise, uses uniform lighting.
- `fogDensity`: Scaled by `0.00001f` before use.
- `sun_direction`: Default value `normalize(make_float3(0.3f, 1.f, 0.5f))`.

**Steps:**

1.  **Thread Identification**: Calculate pixel coordinates:
    $$pixel.x = blockIdx.x \times blockDim.x + threadIdx.x$$
    $$pixel.y = blockIdx.y \times blockDim.y + threadIdx.y$$
    Return immediately if the pixel is outside the image bounds.

2.  **Ray Setup**: Load the ray origin from `params.pathsSurface` using `Path::load`. Compute the camera ray direction:
    $$rayDir = normalize(params.rayMin + pixel.x \times params.rayDDx + (imageHeight - 1 - pixel.y) \times params.rayDDy - params.cameraPosition)$$

3.  **Edge Case Handling**: If `length(rayOrigin) == 0.0f`, set the color with light factor `1.0`, distance `1e9`, and direction `rayDir`, then return.

4.  **Shadow Start Determination**:
    - If `EXACT_SHADOWS` or `PER_VOXEL_FACE_SHADING` is true, compute `hitPosition` using `ray_box_intersection` with `rayOriginDouble` and `rayOriginDouble + 1`.
    - If `EXACT_SHADOWS` is true, set `shadowStart = hitPosition`.
    - Otherwise, set `shadowStart = rayOrigin`.

5.  **Shadow Test**: Get the sun direction. Check if the ray is shadowed:
    $$isShadowed = intersect\_ray\_node\_out\_of\_order(dag, shadowStart + params.shadowBias \times direction, direction)$$
    Where `direction` is the sun direction.

6.  **Lighting and Distance Calculation**:
    - Calculate distance vector: $v = make\_double3(rayOrigin) - params.cameraPosition$.
    - Distance: $distance = length(v)$.
    - Normalized view direction: $nv = v / distance$.

7.  **Color Computation**:
    - If `isShadowed`: Call `setColor(0, distance, nv)`.
    - Else if `PER_VOXEL_FACE_SHADING`:
        - Compute normal from voxel origin to hit position.
        - Truncate normal to integer components.
        - Light factor: $light = max(0.f, dot(make\_float3(normal), sun\_direction()))$.
        - Call `setColor(light, distance, nv)`.
    - Else: Call `setColor(1, distance, nv)`.

8.  **Final Color Application (inside `setColor`)**:
    - Read existing color from `params.colorsSurface`.
    - Apply lighting factor: $lightFactor = clamp(0.5f + light, 0.f, 1.f)$.
    - Apply fog:
        $$fogDensity_{scaled} = fogDensity \times 0.00001f$$
        $$fogAmount = 1.0 - exp(-distance \times fogDensity_{scaled})$$
        $$sunAmount = 1.01f \times max(dot(rayDir, make\_double3(sun\_direction())), 0.0)$$
        $$fogColor = lerp(make\_float3(187, 242, 250) / 255.f, make\_float3(1.0f), float(pow(sunAmount, 30.0)))$$
        $$finalColor = lerp(rgb \times lightFactor, fogColor, clamp(float(fogAmount), 0.f, 1.f))$$
    - Write `finalColor` to `params.colorsSurface` as `uint32` RGB888.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_shadows_shadowing(blockIdx, blockDim, threadIdx, imageHeight, params, dag, sun_direction):
    # 1. Thread Identification
    pixel_x = blockIdx.x * blockDim.x + threadIdx.x
    pixel_y = blockIdx.y * blockDim.y + threadIdx.y
    
    if pixel_x >= image_width or pixel_y >= imageHeight:
        return

    # 2. Ray Setup
    path = Path.load(params.pathsSurface, pixel_x, pixel_y)
    ray_origin = path.rayOrigin
    
    # Compute camera ray direction
    # Note: rayMin, rayDDx, rayDDy are assumed to be accessible from params
    raw_dir = params.rayMin + pixel_x * params.rayDDx + (imageHeight - 1 - pixel_y) * params.rayDDy - params.cameraPosition
    ray_dir = normalize(raw_dir)

    # 3. Edge Case Handling
    if length(ray_origin) == 0.0f:
        color = compute_color(1.0, 1e9, ray_dir, params, sun_direction)
        write_color(params.colorsSurface, pixel_x, pixel_y, color)
        return

    # 4. Shadow Start Determination
    ray_origin_double = make_double3(ray_origin)
    
    if EXACT_SHADOWS or PER_VOXEL_FACE_SHADING:
        hit_position = ray_box_intersection(ray_origin_double, ray_origin_double + 1.0)
        if EXACT_SHADOWS:
            shadow_start = hit_position
        else:
            shadow_start = ray_origin
    else:
        shadow_start = ray_origin

    # 5. Shadow Test
    direction = sun_direction
    # intersect_ray_node_out_of_order is a helper provided by the HashDAG traversal context
    is_shadowed = intersect_ray_node_out_of_order(dag, shadow_start + params.shadowBias * direction, direction)

    # 6. Lighting and Distance Calculation
    v = make_double3(ray_origin) - params.cameraPosition
    distance = length(v)
    nv = v / distance

    # 7. Color Computation
    if is_shadowed:
        light = 0.0
    elif PER_VOXEL_FACE_SHADING:
        # Compute normal from voxel origin to hit position
        # Note: hit_position is only valid if PER_VOXEL_FACE_SHADING is true
        normal_vec = hit_position - ray_origin_double
        normal_int = truncate_to_integer(normal_vec)
        light = max(0.0f, dot(make_float3(normal_int), sun_direction))
    else:
        light = 1.0

    color = compute_color(light, distance, nv, params, sun_direction)
    write_color(params.colorsSurface, pixel_x, pixel_y, color)

def compute_color(light, distance, nv, params, sun_direction):
    # 8. Final Color Application
    # Read existing color (assumed to be background or accumulated color)
    rgb = read_color(params.colorsSurface, ...) # Placeholder for actual read logic
    
    # Apply lighting factor
    light_factor = clamp(0.5f + light, 0.0f, 1.0f)
    
    # Fog calculation
    fog_density_scaled = params.fogDensity * 0.00001f
    fog_amount = 1.0 - exp(-distance * fog_density_scaled)
    
    sun_amount = 1.01f * max(dot(nv, make_double3(sun_direction)), 0.0)
    
    # Fog color interpolation
    # make_float3(187, 242, 250) / 255.f is approximately (0.733, 0.949, 0.980)
    fog_color_base = make_float3(187, 242, 250) / 255.0f
    fog_color_top = make_float3(1.0f)
    fog_color = lerp(fog_color_base, fog_color_top, float(pow(sun_amount, 30.0)))
    
    final_color = lerp(rgb * light_factor, fog_color, clamp(float(fog_amount), 0.0f, 1.0f))
    
    return final_color

def write_color(buffer, x, y, color):
    # Convert float3 to uint32 RGB888
    r = int(clamp(color.x, 0.0, 1.0) * 255)
    g = int(clamp(color.y, 0.0, 1.0) * 255)
    b = int(clamp(color.z, 0.0, 1.0) * 255)
    uint32_color = (r << 16) | (g << 8) | b
    buffer[x, y] = uint32_color
```

## Operating characteristics
- **Time Complexity**: Dominated by the shadow ray intersection `intersect_ray_node_out_of_order`. The complexity depends on the HashDAG depth and the number of nodes traversed by the shadow ray.
- **Memory Complexity**: Minimal additional memory per thread; relies on the HashDAG structure in global memory.
- **Strengths**: Efficiently handles shadows for sparse volumes without explicit geometry reconstruction.
- **Limitations**: Shadow ray tracing can be expensive if the HashDAG is deep or if many rays are shadowed. The `EXACT_SHADOWS` mode requires additional ray-box intersection calculations.
- **When to use**: When shadows are required for sparse voxel volumes and the HashDAG structure is already available for traversal.
- **When not to use**: When real-time performance is critical and shadow rays are too expensive, or when the volume is dense and standard ray marching is more appropriate.

## Background
This method is part of the HashDAG rendering pipeline, specifically designed for GPU-based volume rendering using hierarchical sparse voxel structures. It leverages the HashDAG's ability to represent complex geometry implicitly for shadow testing.

## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
