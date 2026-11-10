# CPU Volume Raycasting for Intersection

Also called: RaycastScene, Volume Raycasting for Intersection, VolumeRaycaster

## Overview
This method performs a CPU-based ray-primitive intersection test against a scene containing multiple volume datasets, returning the closest hit between a world-space ray and any volume object [S1, unofficial]. It differs from GPU-based raycasting by executing the iteration and sampling logic on the CPU, and from simple bounding-box tests by performing actual voxel sampling and transfer function evaluation to determine visibility [S1, unofficial]. It serves as the foundational intersection primitive for CPU-side volume inspection and slicing tools.

## How it works
**Inputs and Outputs**
- **Input**: A world-space ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a scene graph containing instances of `VolumeRenderedObject`.
- **Output**: A `RaycastHit` structure containing the closest `VolumeRenderedObject`, the intersection point in world space, and the distance from the ray origin. Returns false if no volume is hit.

**Data Structures**
The hit result is stored in a `RaycastHit` struct with the following fields [S1, unofficial]:
- `VolumeRenderedObject volumeObject`: Reference to the hit volume object.
- `Vector3 point`: The 3D intersection point in world coordinates.
- `float distance`: The Euclidean distance from the ray origin to the hit point.

**Algorithm Steps**
1. **Scene Traversal**: Retrieve all active `VolumeRenderedObject` instances in the scene [S1, unofficial].
2. **Collection**: For each object, invoke `RaycastObject` with the world-space ray. Collect all successful hits into a list.
3. **Sorting**: Sort the list of hits by `distance` in ascending order.
4. **Selection**: Return the first hit in the sorted list if the list is not empty; otherwise, return failure.

**Per-Object Intersection (`RaycastObject`)**
For a specific volume object, the intersection logic proceeds as follows [S1, unofficial]:

1. **Transform Setup**: Retrieve the volume's local-to-world transform and its cross-section (cutout) data.
2. **Visibility Window**: Calculate the minimum and maximum valid data values based on the dataset's min/max and user-defined visibility window parameters.
3. **Local Space Conversion**: Transform the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ from world space to the volume's local space using the inverse of the volume's transform matrix. Let the local ray be $\mathbf{o}'$ and $\mathbf{d}'$.
4. **AABB Intersection**: Define a unit axis-aligned bounding box (AABB) centered at the origin with bounds $[0, 1]^3$ (i.e., `Vector3.zero` to `Vector3.one`) [S1, unofficial]. Intersect the local ray $\mathbf{o}' + t\mathbf{d}'$ with this AABB to find the entry parameter $t_{start}$.
5. **Raymarch Initialization**: Initialize the raymarch position and direction. If the ray does not intersect the AABB, return false.
6. **Sampling Loop**: Iterate along the ray direction within the AABB bounds:
    a. **Cutout Test**: Check if the current world-space position is occluded by any cross-section/cutout object (e.g., planes, boxes). If hidden, skip sampling for this step.
    b. **Coordinate Conversion**: Convert the current world-space position to texture coordinates $(u, v, w)$ by adding an offset of $0.5$ to the normalized local coordinates.
    c. **Index Calculation**: Calculate the integer indices $(i, j, k)$ for the volume dataset based on the texture coordinates.
    d. **Data Sampling**: Sample the raw data value $V$ at index $(i, j, k)$.
    e. **Normalization**: Normalize $V$ to the range $[0.0, 1.0]$ using the dataset's global min and max values: $V_{norm} = \frac{V - V_{min}}{V_{max} - V_{min}}$.
    f. **Transfer Function**: Evaluate the transfer function alpha $\alpha$ for $V_{norm}$.
    g. **Hit Condition**: If $V_{norm}$ is within the visibility window and $\alpha > 0$, a hit is recorded.
7. **Result Construction**: Transform the hit position back to world space. Calculate the distance as the magnitude of the vector from the original ray origin to the hit point. Return true with the hit data.

**Cutout Evaluation (`IsCutout`)**
The `IsCutout` function determines if a point is inside or outside specific geometric shapes (Plane, BoxInclusive, BoxExclusive) [S1, unofficial]:
- **Plane**: Uses the plane's normal and distance from origin.
- **BoxInclusive/BoxExclusive**: Uses the box's transformation matrix to transform the point into local box space and checks if coordinates lie within $[-0.5, 0.5]$ (or $[0, 1]$ depending on convention) bounds.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_volume_raycast_scene(world_ray_origin, world_ray_direction, scene_objects):
    """
    Finds the closest volume intersection in a scene.
    """
    hits = []
    
    # 1. Retrieve all volume objects
    volume_objects = find_objects_of_type("VolumeRenderedObject", scene_objects)
    
    # 2. Iterate and collect hits
    for obj in volume_objects:
        hit = raycast_object(world_ray_origin, world_ray_direction, obj)
        if hit is not None:
            hits.append(hit)
            
    # 3. Sort by distance
    hits.sort(key=lambda h: h.distance)
    
    # 4. Return closest
    if hits:
        return hits[0]
    return None

def raycast_object(world_ray_origin, world_ray_direction, volume_obj):
    """
    Tests intersection with a single volume object.
    """
    # 1. Get transform and cutouts
    transform = volume_obj.transform
    cutouts = volume_obj.cross_section_data
    
    # 2. Visibility window params
    v_min = volume_obj.dataset_min
    v_max = volume_obj.dataset_max
    vis_window = volume_obj.visibility_window
    
    # 3. Transform ray to local space
    local_ray_origin = transform.world_to_local(world_ray_origin)
    local_ray_direction = transform.world_to_local_direction(world_ray_direction)
    
    # 4. Intersect with unit AABB [0,1]^3
    t_start = intersect_ray_aabb(local_ray_origin, local_ray_direction, 
                                 min_bound=(0,0,0), max_bound=(1,1,1))
    
    if t_start is None:
        return None
        
    # 5. Initialize raymarch
    current_pos = local_ray_origin + local_ray_direction * t_start
    step_size = 1.0 / volume_obj.resolution # Not given in sources, assumed uniform step
    
    # 6. Raymarch loop
    while t_start < 1.0: # Simplified bound check
        # a. Convert to world space for cutout test
        world_pos = transform.local_to_world(current_pos)
        
        # b. Check cutouts
        if not is_occluded(world_pos, cutouts):
            # c. Convert world pos to texture coords (add 0.5 offset)
            local_norm = current_pos # Already in [0,1] if within AABB
            uvw = local_norm + 0.5
            
            # d. Calculate integer indices
            i, j, k = texture_coords_to_indices(uvw, volume_obj.resolution)
            
            # e. Sample data
            raw_value = volume_obj.data_sample(i, j, k)
            
            # f. Normalize
            if v_max != v_min:
                normalized = (raw_value - v_min) / (v_max - v_min)
            else:
                normalized = 0.0
                
            # g. Transfer function check
            alpha = volume_obj.transfer_function(normalized)
            
            if v_min <= normalized <= v_max and alpha > 0:
                # Hit found
                hit_point_world = transform.local_to_world(current_pos)
                distance = vector_magnitude(world_ray_origin - hit_point_world)
                return RaycastHit(volume_obj, hit_point_world, distance)
        
        # Advance ray
        t_start += step_size
        current_pos = local_ray_origin + local_ray_direction * t_start
        
    return None

def is_occluded(world_pos, cutouts):
    """
    Checks if position is inside any cutout shape.
    """
    for cutout in cutouts:
        if cutout.type == "Plane":
            # Check if point is on the "inside" side of the plane
            if point_in_halfspace(world_pos, cutout.plane_normal, cutout.plane_distance):
                return True
        elif cutout.type in ["BoxInclusive", "BoxExclusive"]:
            # Transform to local box space
            local_pos = cutout.transform.world_to_local(world_pos)
            if is_inside_box(local_pos, cutout.bounds):
                return True
    return False
```

## Operating characteristics
- **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of volume objects and $M$ is the number of samples per ray. Sorting hits adds $O(K \log K)$ where $K$ is the number of hits.
- **Memory Complexity**: $O(K)$ for storing hit results.
- **Performance**: Significantly slower than GPU-based raycasting due to CPU cache misses and lack of SIMD/GPU parallelism [S1, unofficial]. Suitable for low-resolution inspection or interactive slicing tools where latency is critical but frame rate is low.
- **Strengths**: Simple implementation, easy to debug, integrates directly with CPU-side scene graphs (e.g., Unity GameObjects).
- **Limitations**: Poor scalability with high-resolution volumes or many objects. No hardware acceleration for ray traversal.

## Background
This method originates from the `UnityVolumeRendering` project by mlavik1, an unofficial implementation of volume rendering in Unity [S1, unofficial]. It is designed for interactive tools within the Unity engine, leveraging its component-based architecture (`VolumeRenderedObject`, `SlicingPlane`). It is not a peer-reviewed academic algorithm but a practical engineering solution for CPU-side volume interaction in game engines.

## Sources
[S1] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
