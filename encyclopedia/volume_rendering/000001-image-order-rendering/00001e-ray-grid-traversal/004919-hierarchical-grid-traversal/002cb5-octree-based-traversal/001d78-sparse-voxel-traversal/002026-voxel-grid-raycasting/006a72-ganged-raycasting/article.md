# Ganged Raycasting

Also called: Horizontal Ganging, Ganged Tracing, Rectangular Pyramid Tracing

## Overview
Ganged Raycasting is a volume rendering optimization that traces multiple rays simultaneously as a group (gang) to reduce per-ray overhead, specifically by treating a rectangular pyramid of rays as a single unit rather than processing them individually [S1, unofficial]. It distinguishes itself from standard Voxel Grid Raycasting by batching rays that share the same y-coordinate and initial voxel intersection, splitting the group only when the ray set diverges into non-continuous voxel sets [S1, unofficial]. This approach leverages the typical aspect ratio of images (wider than tall) to maximize the number of rays starting in the same voxel, thereby reducing the total number of traversal steps required [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A voxel grid, camera parameters (origin, direction vectors), and image dimensions $(W, H)$.
- Output: Rendered image buffer.

**Data Structures:**
- `RayGang`: A structure representing a group of rays.
  - `y_start`: Integer, the shared y-coordinate of the rays in the gang.
  - `x_start`: Integer, the starting x-coordinate of the first ray in the horizontal span.
  - `x_end`: Integer, the ending x-coordinate of the last ray in the horizontal span.
  - `origin`: Vector3, the camera origin (shared for all rays in the initial trace).
  - `direction`: Vector3, the base direction vector for the first ray (used to derive others).
  - `xnext`, `ynext`, `znext`: Vectors of floats, storing the distance to the next voxel face intersection for the start and end rays of the gang (and potentially intermediate rays if needed for divergence detection). *Note: The source implies storing intersection points in vectors `xnext`, `ynext`, and `znext` [S1, unofficial]. For a horizontal gang, the y-dimension is constant, so `ynext` is likely trivial or unused for divergence, but stored for consistency with the general tracer logic.*

**Numbered Steps:**
1. **Initialization**: For each y-coordinate from $0$ to $H-1$:
   - Identify the range of x-coordinates $[x_{start}, x_{end}]$ such that all rays with these coordinates intersect the voxel grid at the same initial voxel. This is determined by casting the first ray at $(x_{start}, y)$ and the last at $(x_{end}, y)$ and verifying they hit the same voxel, and that the span is continuous [S1, unofficial].
   - Create a `RayGang` with `y_start = y`, `x_start`, `x_end`, and the corresponding direction vectors.
   - Calculate the distance to the first voxel face intersection for the rays in the group. Store these in `xnext`, `ynext`, and `znext` vectors. For a horizontal gang, `ynext` is constant or irrelevant for splitting, but `xnext` and `znext` vary between the left and right edges of the pyramid [S1, unofficial].

2. **Ganged Traversal Loop**:
   - While the `RayGang` is valid (i.e., $x_{start} \le x_{end}$):
     - Determine the minimum distance to the next voxel face among all rays in the gang. This is typically derived from the `xnext` and `znext` values of the boundary rays, assuming the pyramid expands monotonically [S1, unofficial].
     - Advance all rays in the gang by this minimum distance.
     - Update the voxel occupancy for the current position of all rays.
     - Check for divergence: The group remains valid as long as all rays stay within the same voxel or neighboring voxels [S1, unofficial].
     - **Split Condition**: If the dot product of the ray's current position integer coordinates exceeds 1, the group must be split [S1, unofficial]. *Note: The source phrasing "dot product of the ray current position integer coordinates exceeds 1" is ambiguous. In standard grid traversal, this likely refers to a check on the integer grid coordinates $(i, j, k)$ of the ray's position. A common interpretation in ganged tracing is checking if the rays have moved past a voxel boundary in a way that breaks the "neighboring" constraint. Given the context of splitting a pyramid, it likely means checking if the integer coordinates of the start and end rays differ by more than a threshold (e.g., 1) in any dimension, or if the ray has exited the current voxel cluster.* [G: Assuming the standard interpretation, the split condition is when the integer grid coordinates of the leftmost and rightmost rays in the gang differ by more than 1 in any dimension, indicating they are no longer in neighboring voxels.]
     - **Splitting**:
       - Calculate the split point's x-coordinate. Use this x-coordinate as a weight to determine the new split ray positions [S1, unofficial].
       - Split the `RayGang` into two new `RayGang` instances: one for $[x_{start}, x_{split}]$ and one for $[x_{split}+1, x_{end}]$.
       - Recalculate `xnext`, `ynext`, `znext` for the new gangs.
       - Add the new gangs to the processing queue.

3. **Termination**:
   - When a gang reduces to a single ray ($x_{start} == x_{end}$), trace it individually to the end of the volume or until it exits the grid.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ganged_raytrace(voxel_grid, camera, width, height):
    # Initialize output buffer
    image = zeros((height, width, 3))
    
    # Queue for active ray gangs
    gang_queue = []
    
    # Step 1: Initialize gangs
    for y in range(height):
        # Find continuous x-ranges that start in the same voxel
        x = 0
        while x < width:
            # Cast ray at (x, y) to find initial voxel
            initial_voxel = get_voxel_at(camera, x, y)
            if initial_voxel is None:
                x += 1
                continue
            
            # Find the end of the continuous range
            x_end = x
            while x_end + 1 < width:
                next_voxel = get_voxel_at(camera, x_end + 1, y)
                if next_voxel == initial_voxel:
                    x_end += 1
                else:
                    break
            
            # Create gang
            gang = {
                'y': y,
                'x_start': x,
                'x_end': x_end,
                'origin': camera.origin,
                'dir_start': camera.get_direction(x, y),
                'dir_end': camera.get_direction(x_end, y),
                'xnext': [0.0, 0.0], # Distances for start and end rays
                'ynext': [0.0, 0.0],
                'znext': [0.0, 0.0]
            }
            
            # Calculate initial intersection distances
            gang['xnext'], gang['ynext'], gang['znext'] = calculate_first_intersection(gang, voxel_grid)
            
            gang_queue.append(gang)
            x = x_end + 1

    # Step 2: Process gangs
    while gang_queue:
        gang = gang_queue.pop(0)
        
        # If single ray, trace individually
        if gang['x_start'] == gang['x_end']:
            trace_single_ray(image, camera, gang['y'], gang['x_start'], voxel_grid)
            continue
        
        # Determine minimum step size
        # Assuming xnext and znext are stored for start and end rays
        min_dist = min(gang['xnext'][0], gang['xnext'][1], gang['znext'][0], gang['znext'][1])
        
        # Advance rays
        # Note: In a pyramid, the step size is limited by the ray that hits a face first
        # This logic assumes a simplified model where we advance by the min distance
        # and then re-evaluate.
        
        # Check for split condition
        # Source: "dot product of the ray current position integer coordinates exceeds 1"
        # Interpretation: Check if integer coordinates of start and end rays differ significantly
        pos_start = get_ray_position(gang, min_dist)
        pos_end = get_ray_position(gang, min_dist) # Both advance by same dist in this simplified step
        
        int_start = floor(pos_start)
        int_end = floor(pos_end)
        
        # Heuristic for split: if integer coordinates differ by more than 1 in any axis
        if abs(int_start[0] - int_end[0]) > 1 or abs(int_start[1] - int_end[1]) > 1 or abs(int_start[2] - int_end[2]) > 1:
            # Split the gang
            # Use split point x-coordinate as weight
            # This part is ambiguous in source. We assume a binary split at the midpoint
            # or based on the divergence point.
            split_x = (gang['x_start'] + gang['x_end']) // 2
            
            # Create two new gangs
            gang_left = copy(gang)
            gang_left['x_end'] = split_x
            gang_left['xnext'], gang_left['ynext'], gang_left['znext'] = calculate_first_intersection(gang_left, voxel_grid)
            
            gang_right = copy(gang)
            gang_right['x_start'] = split_x + 1
            gang_right['xnext'], gang_right['ynext'], gang_right['znext'] = calculate_first_intersection(gang_right, voxel_grid)
            
            gang_queue.append(gang_left)
            gang_queue.append(gang_right)
        else:
            # Advance and accumulate color
            accumulate_color(image, gang, min_dist, voxel_grid)
            # Update intersection distances for next step
            gang['xnext'], gang['ynext'], gang['znext'] = update_intersection_distances(gang, min_dist, voxel_grid)
            gang_queue.append(gang)

def calculate_first_intersection(gang, voxel_grid):
    # Calculate distances to next voxel face for start and end rays
    # Returns (xnext, ynext, znext) for start and end
    pass

def update_intersection_distances(gang, dist, voxel_grid):
    # Update distances after advancing by 'dist'
    pass

def get_ray_position(gang, dist):
    # Return world position of the ray at distance 'dist'
    pass

def trace_single_ray(image, camera, y, x, voxel_grid):
    # Standard voxel raycasting for a single ray
    pass

def accumulate_color(image, gang, dist, voxel_grid):
    # Accumulate color for the rays in the gang
    pass
```

**Parameters:**
- Typical image aspect ratio: Images are much wider than they are tall, supporting horizontal ganging [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Split Ray Precision**: The new split ray may not land exactly on a voxel face after splitting, requiring careful handling of floating-point precision and ray origin adjustments [S1, unofficial].
- **Divergence Detection**: The condition for splitting must accurately detect when rays are no longer in neighboring voxels. The source's mention of "dot product... exceeds 1" is unclear and may require empirical tuning or a more robust geometric check [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Theoretically faster than individual tracing because it saves computation by treating many rays starting at the same voxel(s) as a single rectangular pyramid trace [S1, unofficial]. The speedup depends on the number of rays that can be ganged, which is high in scenes with large empty spaces or uniform voxel distributions.
- **Memory Complexity**: Requires storing additional data structures for the gangs (`xnext`, `ynext`, `znext` vectors) and a queue for active gangs. Memory usage is proportional to the number of active gangs, which is typically much less than the number of rays.
- **Measured Performance**: In a 1920x1080 projection, there are approximately 2,073,600 rays, but they intersect under 200 initial voxels, meaning over 10,000 rays start at the same voxel [S1, unofficial]. This suggests a significant potential for speedup, though the actual gain depends on the splitting overhead and the efficiency of the ganged traversal loop.
- **Strengths**: Reduces per-ray overhead, leverages spatial coherence, and is well-suited for wide images.
- **Limitations**: Splitting logic can be complex and may introduce precision errors. Not as effective for tall, narrow images or scenes with high voxel density near the camera.

## Background
Ganged Raycasting was discussed in the context of voxel raycasting algorithms on GameDev Stack Exchange in April 2021 [S1, unofficial]. It is presented as a theoretical optimization for horizontal ganging, comparing its potential speedup to individual ray tracing. The method is not associated with a specific peer-reviewed paper in the provided sources but is described as a technique for optimizing ray tracing in voxel grids by grouping rays.

## Sources
[S1] Voxel raycasting algorithm not working correctly, https://gamedev.stackexchange.com/questions/191245/voxel-raycasting-algorithm-not-working-correctly, fetched 2026-10-27 (unofficial)
