# Voxel Shadow Raycasting with DDA

Also called: DDA Voxel Shadow Tracing, Shadow Ray Grid Optimization, Voxel Shadow Raytracer

## Overview
Voxel Shadow Raycasting with DDA is a technique for computing hard shadows in voxel-based scenes by traversing a 3D voxel grid along rays cast from surface points to light sources [S1, unofficial]. It solves the problem of efficiently determining light attenuation through volumetric or opaque voxel data without resorting to slow, fixed-step ray marching [S1, unofficial]. It sets itself apart from its siblings by focusing specifically on the shadow calculation phase within a rendering pipeline, often utilizing the Amanatides–Woo traversal logic to jump between grid cells rather than sampling every unit of distance [S1, unofficial].

## How it works
The method operates in two distinct phases: the main rendering loop (which invokes the shadow raycaster) and the DDA traversal algorithm itself. It relies on a sparse or dense voxel buffer where each voxel has a density value [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   **Voxel Grid:** A buffer of `voxel_material` structs. Each struct contains `color` (unused in shadow calc) and `density` (float, 0.0 to 1.0, where 1.0 is opaque) [S1, unofficial].
*   **Light Sources:** A buffer of `lightsource` structs containing `position` (vec3), `power` (float, max reach distance), `color` (vec3), and `intensity` (float) [S1, unofficial].
*   **Surface Data:** For each pixel, the world-space `position` (vec4, alpha > 0 indicates valid surface) and `normal` (vec3) [S1, unofficial].
*   **Grid Dimensions:** `map_size` (vec3), defining the bounds of the voxel world [S1, unofficial].

**Outputs:**
*   **Shadow Factor:** A float value between 0.0 (fully shadowed) and 1.0 (fully lit), representing the remaining light intensity after attenuation [S1, unofficial].
*   **Final Color:** The base pixel color multiplied by the accumulated lighting contributions [S1, unofficial].

### Data Structures
The implementation uses specific buffer layouts, typically in a GLSL compute shader context [S1, unofficial]:

1.  **Voxel Map (`Map`):** `layout (std430, binding = 3) buffer Map` containing `voxel_material voxels[]`.
2.  **Lights (`Lights`):** `layout (std430, binding = 2) buffer Lights` containing `lightsource sources[]`.
3.  **Position Texture (`pos_texture`):** `layout(rgba32f, binding = 1)` storing world-space positions.
4.  **Normal Texture (`norm_texture`):** `layout(rgba32f, binding = 4)` storing voxel normals.
5.  **Output Image (`img_output`):** `layout(rgba32f, binding = 0)` storing the final lit color.

**Coordinate to ID Conversion:**
To access the voxel buffer, 3D coordinates are converted to a 1D index. The coordinates are clamped to `[0, map_size]` [S1, unofficial]:
$$ \text{ID} = \lfloor x \rfloor + \lfloor y \rfloor \cdot \text{map\_size}.x + \lfloor z \rfloor \cdot \text{map\_size}.x \cdot \text{map\_size}.y $$
This assumes a row-major layout where $x$ is the fastest varying dimension [S1, unofficial].

### Algorithm Steps

#### 1. Main Rendering Loop
For each pixel $(x, y)$:
1.  Load `pixel` color, `position`, and `normal` from textures.
2.  If `position.a <= 0`, skip (background) [S1, unofficial].
3.  Initialize `lightValue` to ambient color `vec4(0.2, 0.2, 0.2, 1)` and `lightColor` to `vec3(0,0,0)` [S1, unofficial].
4.  Iterate through each light source `current` in the `Lights` buffer:
    a.  Calculate squared distance `distanceSP = distance(current.position, position.xyz)` [S1, unofficial].
    b.  **Light Reach Test:** If `distanceSP > current.power * current.power`, skip this light (it cannot reach the surface) [S1, unofficial].
    c.  Compute ray direction `rayPixSrc = current.position - position.xyz` [S1, unofficial].
    d.  Determine if the starting voxel should be sampled: `stepThroughFirstVoxel = dot(normalize(rayPixSrc), normal) > 0`. This checks if the ray originates from the surface and potentially exits it immediately [S1, unofficial].
    e.  Call `stepThroughVoxelWorld(position.xyz, rayPixSrc, stepThroughFirstVoxel)` to get `intensity` [S1, unofficial].
    f.  Calculate result multiplier: `result = current.intensity * intensity * (1 - (distanceSP / current.power))` [S1, unofficial].
    g.  If `result > 0`, add `current.color.rgb * result` to `lightColor` [S1, unofficial].
5.  Add `lightColor` to `lightValue.rgb` [S1, unofficial].
6.  Store `pixel * lightValue` to `img_output` [S1, unofficial].

#### 2. DDA Traversal (`stepThroughVoxelWorld`)
This function traverses the voxel grid from `origin` along `dir` [S1, unofficial].

**Initialization Phase:**
1.  Normalize `dir` [S1, unofficial].
2.  Set `currPos = ivec3(floor(origin.x), floor(origin.y), floor(origin.z))` [S1, unofficial].
3.  Set `intensity = 1.0` [S1, unofficial].
4.  For each axis $i \in \{x, y, z\}$:
    a.  If `dir[i] >= 0`:
        *   `step[i] = 1`
        *   `delta[i] = 1.0 / max(0.000001, dir[i])` (prevents division by zero) [S1, unofficial].
        *   `Max[i] = (floor(currPos[i] + 1) - origin[i]) / dir[i]`
        *   `Out[i] = map_size[i]` (boundary limit) [S1, unofficial].
    b.  If `dir[i] < 0`:
        *   `step[i] = -1`
        *   `delta[i] = -1.0 / dir[i]`
        *   `Max[i] = (floor(currPos[i]) - origin[i]) / dir[i]`
        *   `Out[i] = -1` (boundary limit) [S1, unofficial].
5.  Calculate `last = ivec3(floor(origin.x + dir.x), floor(origin.y + dir.y), floor(origin.z + dir.z))` [S1, unofficial].
    *   *Note:* The source also calculates `maxLength = dir.x^2 + dir.y^2 + dir.z^2`, but since `dir` is normalized, this is always 1.0. The termination condition primarily relies on `currPos != last` [S1, unofficial].

**Step Loop:**
While `intensity > 0` AND `!reachedEnd` AND `currPos != last`:
1.  Sample voxel at `currPos` using `CordtoID`.
2.  Subtract `currentVoxel.density` from `intensity` [S1, unofficial].
3.  Determine the next axis to step: Find the axis $k$ with the minimum `Max[k]` [S1, unofficial].
4.  Increment `currPos[k]` by `step[k]` [S1, unofficial].
5.  Update `Max[k] += delta[k]` [S1, unofficial].
6.  Check boundary: If `currPos[k] == Out[k]`, set `reachedEnd = true` [S1, unofficial].

**Return:**
Return `intensity` [S1, unofficial].

### Optimization: Shadow Bit Caching
To reduce computation, shadow status can be stored in the voxel grid [S2, unofficial]. For each voxel and each light source, two bits are stored:
1.  One bit indicating the voxel is completely in shadow.
2.  One bit indicating nothing blocks the voxel from seeing the light source [S2, unofficial].
During traversal, if these bits are set, the ray calculation is skipped [S2, unofficial]. This adds a memory overhead of 2 bits per light source per voxel [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
AMBIENT = vec4(0.2, 0.2, 0.2, 1.0)
EPSILON = 0.000001

def cord_to_id(cord, map_size):
    # Clamp coordinates to [0, map_size]
    x = max(0, min(map_size.x, cord.x))
    y = max(0, min(map_size.y, cord.y))
    z = max(0, min(map_size.z, cord.z))
    # Integer floor and linear index calculation
    return int(floor(x) + floor(y) * map_size.x + floor(z) * map_size.x * map_size.y)

def step_through_voxel_world(origin, dir, step_through_first_voxel, map_size, voxels):
    intensity = 1.0
    
    # Initialization
    dir_norm = normalize(dir)
    curr_pos = ivec3(floor(origin.x), floor(origin.y), floor(origin.z))
    
    # Arrays for DDA state
    step = ivec3(0, 0, 0)
    delta = vec3(0.0, 0.0, 0.0)
    max_dist = vec3(0.0, 0.0, 0.0)
    out_bound = ivec3(0, 0, 0)
    
    # Configure DDA for each axis
    for axis in ['x', 'y', 'z']:
        d = dir_norm[axis]
        if d >= 0:
            step[axis] = 1
            delta[axis] = 1.0 / max(EPSILON, d)
            max_dist[axis] = (floor(curr_pos[axis] + 1) - origin[axis]) / d
            out_bound[axis] = int(map_size[axis])
        else:
            step[axis] = -1
            delta[axis] = -1.0 / d
            max_dist[axis] = (floor(curr_pos[axis]) - origin[axis]) / d
            out_bound[axis] = -1
            
    # Termination setup
    last_pos = ivec3(floor(origin.x + dir_norm.x), 
                     floor(origin.y + dir_norm.y), 
                     floor(origin.z + dir_norm.z))
    
    reached_end = False
    
    # If we need to sample the starting voxel, do it now
    if step_through_first_voxel:
        voxel_idx = cord_to_id(vec3(curr_pos), map_size)
        intensity -= voxels[voxel_idx].density
        
    # Main Loop
    while intensity > 0 and not reached_end and curr_pos != last_pos:
        # Determine next axis to step
        # Find index of minimum value in max_dist
        min_val = min(max_dist.x, max_dist.y, max_dist.z)
        if max_dist.x == min_val:
            next_axis = 'x'
        elif max_dist.y == min_val:
            next_axis = 'y'
        else:
            next_axis = 'z'
            
        # Step
        curr_pos[next_axis] += step[next_axis]
        max_dist[next_axis] += delta[next_axis]
        
        # Check boundary
        if curr_pos[next_axis] == out_bound[next_axis]:
            reached_end = True
            break
            
        # Sample voxel
        voxel_idx = cord_to_id(vec3(curr_pos), map_size)
        intensity -= voxels[voxel_idx].density
        
    return intensity

def render_pixel(x, y, pos_texture, norm_texture, img_output, lights, voxels, map_size):
    pixel = image_load(img_output, ivec2(x, y))
    position = image_load(pos_texture, ivec2(x, y))
    normal = image_load(norm_texture, ivec2(x, y)).rgb
    
    if position.a <= 0:
        return
        
    light_value = AMBIENT
    light_color = vec3(0, 0, 0)
    cast_lights = 0
    
    for light in lights:
        dist_sq = length(light.position - position.xyz)
        
        # Light reach test
        if dist_sq > light.power * light.power:
            continue
            
        ray_dir = light.position - position.xyz
        # Check if ray exits the surface immediately
        step_first = dot(normalize(ray_dir), normal) > 0
        
        intensity = step_through_voxel_world(position.xyz, ray_dir, step_first, map_size, voxels)
        
        # Calculate result
        # Note: distanceSP is used in the formula, but dist_sq is squared distance.
        # The source uses distanceSP (linear distance) in the falloff term.
        result = light.intensity * intensity * (1.0 - (dist_sq / light.power)) # Correction: Source uses distanceSP, not dist_sq. 
        # Re-reading source: "float distanceSP = distance(...)". So distanceSP is linear.
        # Let's correct the pseudocode variable name to match source logic exactly.
        
        # Corrected calculation based on source:
        # distanceSP is linear distance.
        # result = current.intensity * intensity * (1 - (distanceSP / current.power))
        
        # Since I defined dist_sq above, let's use linear distance for the formula
        linear_dist = sqrt(dist_sq)
        result = light.intensity * intensity * (1.0 - (linear_dist / light.power))
        
        if result > 0:
            light_color += light.color * result
            cast_lights += 1
            
    if cast_lights > 0:
        light_value.rgb += light_color
        
    image_store(img_output, ivec2(x, y), pixel * light_value)
```

## Subtopics

### [WebGPU Voxel Ray Tracing Renderer](4993-webgpu-voxel-ray-tracing-renderer.md)
A WebGPU compute shader-based ray tracer for voxel models that replaces a rasterizer. It casts rays per pixel, traverses a 3D voxel grid using DDA, applies directional lighting with diffuse shading, and generates hard shadows via shadow rays. It sets itself apart from its parent [Voxel Shadow Raycasting with DDA](2410-voxel-shadow-raycasting-with-dda.md) by implementing the entire rendering pipeline (primary rays, shading, and shadow rays) within a WebGPU compute shader, rather than relying on a GLSL rasterizer with DDA for shadow calculation only.

## Operating characteristics
*   **Time Complexity:** The DDA traversal is $O(N)$ where $N$ is the number of voxels intersected by the shadow ray. This is significantly faster than fixed-step marching for sparse or large grids, as it skips empty space [S1, unofficial].
*   **Memory Complexity:** The base method requires storing the voxel grid. The optimized version with shadow bits requires an additional 2 bits per light source per voxel [S2, unofficial].
*   **Performance:** The DDA approach avoids the "slow fallback" fixed-step algorithm which increments position by a small fixed step (e.g., 0.05) and samples every step, leading to redundant calculations [S1, unofficial].
*   **Strengths:** Efficient traversal of uniform grids; handles arbitrary ray directions; simple implementation [S1, unofficial].
*   **Limitations:** Requires a uniform grid structure; shadow bit optimization increases memory usage linearly with the number of lights [S2, unofficial].
*   **When to use:** When rendering hard shadows in voxel worlds where performance is critical and the grid is uniform.
*   **When not to use:** For non-uniform grids (use [Siddon's Algorithm](2590-siddon-s-algorithm.md) or adaptive grids) or when soft shadows are required (DDA provides hard shadows only).

## Background
The technique is based on the Amanatides–Woo voxel traversal algorithm, which enumerates grid cells intersected by a ray in front-to-back order [S1, unofficial]. The specific GLSL implementation described in the sources is an unofficial contribution to the GitHub repository `Ein-Baum/Voxel-shadow-raytracer` [S1, unofficial]. The concept of using grids to reduce shadow ray computations by storing status bits was discussed in academic literature on fast voxel traversal [S2, unofficial].

## Sources
[S1] Ein-Baum/Voxel-shadow-raytracer, https://github.com/Ein-Baum/Voxel-shadow-raytracer, fetched 2026-10-24 (unofficial)
[S2] Fast Voxel Traversal for Ray Tracing, https://www.scribd.com/document/841721013/Voxel-Traversal-Paper, fetched 2026-10-25 (unofficial)
