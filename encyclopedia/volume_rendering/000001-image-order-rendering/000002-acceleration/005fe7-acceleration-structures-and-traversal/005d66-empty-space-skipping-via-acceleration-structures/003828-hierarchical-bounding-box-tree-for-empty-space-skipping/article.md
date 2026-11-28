# Hierarchical Bounding Box Tree for Empty-Space Skipping

Also called: Accelerated Ray Casting with BVH and Early Termination, Large-Scale Aerosols Visualization via Accelerated Ray Casting, Large-scale Aerosols Visualization Method, Ray Casting Acceleration Algorithm

## Overview
This method accelerates traditional front-to-back ray casting for large-scale volumetric data (specifically aerosols like smoke, fog, and dust) by combining a Hierarchical Bounding Box Tree (BVH) for empty-space culling, early termination of the compositing loop, and adaptive sampling with grouped linear interpolation [S1, S2]. It addresses the low efficiency and high resource consumption of standard ray casting when processing large scalar fields [S1]. Unlike sibling methods that may rely on octrees or sparse grids, this approach uses a BVH to avoid recording numerous empty nodes, thereby reducing computational complexity and allowing for dynamic updates when the dataset changes [S1].

## How it works

### Inputs and Outputs
*   **Input:** A 3D scalar field representing volume data (e.g., density/opacity values for aerosols); camera parameters (viewpoint, direction, field of view); rendering parameters (early termination threshold, sampling intervals) [G].
*   **Output:** A 2D image where each pixel color is computed via front-to-back compositing, skipping empty regions and terminating rays that are sufficiently opaque [G].

### Data Structures

#### Axis-Aligned Bounding Box (AABB)
An AABB is defined by two 3D vectors: `min` and `max`, representing the minimum and maximum coordinates along the x, y, and z axes. The six faces are parallel to the xy, xz, and yz planes [G].

#### Hierarchical Bounding Box Tree (BVH)
The BVH is a binary tree used to store the shape of bounding boxes [S1].
*   **Root Node:** Represents the largest bounding box encompassing the entire volume data [S1].
*   **Internal Nodes:** Each internal node has exactly two child nodes, representing two sub-bounding boxes that partition the parent's space [S1].
*   **Leaf Nodes:** Each leaf node corresponds to a sub-block of the volume data [S1].
*   **Node Attributes:**
    *   **Empty:** The sub-block contains no valid data (or values below a significance threshold) [G].
    *   **Non-Empty:** The sub-block contains valid data [S1].
    *   **Unknown:** The status is not yet determined or requires further traversal (used during construction) [G].
    *   **Proportion:** For non-leaf nodes, an attribute representing the proportion of child nodes that are non-empty, used to assign the node's primary attribute [S1].

#### Construction Details
1.  **Subdivision:** The volume data is subdivided into blocks. Each leaf node corresponds to one such block [S1].
2.  **Boundary Expansion:** To avoid interpolation errors at block boundaries (where interpolated values might exceed the original block's min/max), each sub-data block is expanded by one voxel in all directions during the construction phase [S1].
3.  **Min/Max Storage:** The minimum and maximum values of each sub-data block are counted and stored in the bounding box structure to facilitate 3D texture generation and culling decisions [S1].
4.  **Octree Pre-processing:** The data is recursively divided into eight parts to form an octree. Leaf nodes of this octree are marked as Empty, Non-Empty, or Unknown. Non-leaf nodes are assigned the attribute of their children that has the highest proportion [S1].
5.  **BVH Generation:** A breadth-first search is performed on the octree. Bounding boxes are constructed for volume data blocks whose attributes differ from their parent nodes. These boxes form the BVH [S1].

### Algorithm Steps

#### 1. BVH Construction
1.  Divide the volume data into initial blocks [S1].
2.  For each block, expand the data by one voxel to handle boundary interpolation safety [S1].
3.  Compute the minimum and maximum scalar values for each expanded block [S1].
4.  Build an underlying octree by recursively dividing blocks into eight children [S1].
5.  Label octree leaves as Empty, Non-Empty, or Unknown based on data presence [S1].
6.  For each octree internal node, calculate the proportion of Non-Empty children. Assign the node's attribute to the majority label (e.g., if >50% children are Non-Empty, label the parent Non-Empty) [S1].
7.  Traverse the octree (BFS). Identify blocks where the attribute differs from the parent [S1].
8.  Construct AABBs for these differing blocks [S1].
9.  Organize these AABBs into a binary BVH structure, where internal nodes represent the union of their children's AABBs, and leaves represent the specific data blocks [S1].

#### 2. Ray Casting with Empty-Space Skipping
For each pixel, cast a ray from the camera through the pixel into the volume [G].
1.  **Ray-BVH Intersection:** Determine the entry and exit points of the ray into the BVH [G].
2.  **Traversal:** Traverse the BVH from the root [G].
    *   If the ray does not intersect a node's AABB, skip the entire subtree [G].
    *   If the ray intersects a leaf node's AABB:
        *   Check the leaf's attribute [G].
        *   If **Empty**, skip sampling within this block [S1].
        *   If **Non-Empty**, proceed to sample [S1].
3.  **Merging and Deletion:** As the ray passes through multiple bounding boxes, merge overlapping valid ranges and delete empty ranges to determine the final valid sampling intervals [S1].

#### 3. Adaptive Sampling and Interpolation
Standard ray casting uses a fixed sampling interval. This method adapts the interval and interpolation based on the ray's position and data validity [S1].
1.  **Adaptive Interval:**
    *   Initialize a base sampling interval $\Delta$ [G].
    *   As the ray traverses the volume:
        *   If an **invalid** (empty) sampling point is encountered, increase the sampling interval (reduce resampling frequency) [S1].
        *   If a **valid** (non-empty) sampling point is encountered, restore the interval to the base value [S1].
2.  **Grouped Linear Interpolation:**
    *   Divide all resampling points on the ray into three levels based on their distance from the viewpoint:
        *   **Level 1:** Closest to the viewpoint [S1].
        *   **Level 2:** Intermediate distance [S1].
        *   **Level 3:** Farthest from the viewpoint [S1].
    *   Apply different linear interpolation complexities for each level. The amount of interpolation calculation decreases from Level 1 to Level 3 [S1]. This reduces computational load without significantly reducing image authenticity [S1].

#### 4. Early Termination
During the front-to-back compositing loop, check the cumulative opacity $\alpha_{acc}$ after each sample [G].
1.  **Condition:** If $\alpha_{acc} > T_{term}$, where $T_{term}$ is the early termination threshold, terminate the ray calculation [S1].
2.  **Rationale:** As $\alpha_{acc}$ approaches 1, the pixel value becomes saturated, and further sampling has minimal effect on the final color [S1].
3.  **Threshold:** The specific numerical value for $T_{term}$ is not given in the sources [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
T_TERM = 0.95          # Early termination threshold [S1]
BASE_INTERVAL = 0.5    # Initial sampling step size [G]
LEVEL_1_MAX = 10.0     # Distance threshold for Level 1 [G]
LEVEL_2_MAX = 20.0     # Distance threshold for Level 2 [G]
BACKGROUND_COLOR = [0.0, 0.0, 0.0] # Default background [G]

def ray_aabb_intersection(ray_origin, ray_dir, aabb_min, aabb_max):
    """Returns (t_min, t_max) of ray intersection with AABB, or None."""
    t_min = float('-inf')
    t_max = float('inf')
    for i in range(3):
        if ray_dir[i] == 0:
            if ray_origin[i] < aabb_min[i] or ray_origin[i] > aabb_max[i]:
                return None
        else:
            t1 = (aabb_min[i] - ray_origin[i]) / ray_dir[i]
            t2 = (aabb_max[i] - ray_origin[i]) / ray_dir[i]
            if t1 > t2: t1, t2 = t2, t1
            if t1 > t_min: t_min = t1
            if t2 < t_max: t_max = t2
            if t_min > t_max: return None
    if t_max < 0: return None
    return (t_min if t_min > 0 else 0, t_max)

def merge_ranges(ranges):
    """Merges overlapping intervals."""
    if not ranges: return []
    ranges.sort(key=lambda r: r[0])
    merged = [ranges[0]]
    for start, end in ranges[1:]:
        if start <= merged[-1][1]:
            merged[-1] = (merged[-1][0], max(merged[-1][1], end))
        else:
            merged.append((start, end))
    return merged

def get_valid_ranges_bvh(ray_origin, ray_dir, bvh_root):
    """Traverses BVH to find valid sampling intervals."""
    valid_ranges = []
    stack = [bvh_root]
    while stack:
        node = stack.pop()
        t_min, t_max = ray_aabb_intersection(ray_origin, ray_dir, node.aabb_min, node.aabb_max)
        if t_min is None: continue
        if node.is_leaf:
            if node.attribute == "non_empty":
                valid_ranges.append((t_min, t_max))
        else:
            stack.append(node.right)
            stack.append(node.left)
    return merge_ranges(valid_ranges)

def get_interpolation_level(distance):
    """Divides resampling points into 3 levels based on distance from viewpoint."""
    if distance < LEVEL_1_MAX: return 1
    elif distance < LEVEL_2_MAX: return 2
    else: return 3

def cast_ray(ray_origin, ray_dir, volume_data, bvh_root):
    """Casts a single ray, using BVH for skipping and early termination."""
    valid_ranges = get_valid_ranges_bvh(ray_origin, ray_dir, bvh_root)
    if not valid_ranges:
        return BACKGROUND_COLOR
    
    alpha_acc = 0.0
    color_acc = [0.0, 0.0, 0.0]
    step_size = BASE_INTERVAL
    
    for start_t, end_t in valid_ranges:
        step_size = BASE_INTERVAL
        t = start_t
        while t < end_t:
            sample_pos = [ray_origin[i] + ray_dir[i] * t for i in range(3)]
            sample = volume_data.sample(sample_pos)
            alpha_sample = volume_data.transfer_function(sample)
            color_sample = volume_data.illumination_model(sample, sample_pos, ray_dir)
            
            weight = alpha_sample * (1.0 - alpha_acc)
            color_acc = [color_acc[i] + color_sample[i] * weight for i in range(3)]
            alpha_acc += weight
            
            if alpha_acc > T_TERM:
                break
                
            # Adaptive sampling: increase step on invalid points, restore on valid
            if sample < volume_data.invalid_threshold:
                step_size *= 2.0
            else:
                step_size = BASE_INTERVAL
                
            distance = sum((sample_pos[i] - ray_origin[i])**2 for i in range(3))**0.5
            level = get_interpolation_level(distance)
            # Interpolation complexity handled in volume_data.sample(level)
            
            t += step_size
        if alpha_acc > T_TERM:
            break
            
    return color_acc
```

### Parameters
*   **Early Termination Threshold ($T_{term}$):** A value for cumulative opacity $\alpha_{acc}$ above which the ray calculation is terminated. The sources state "a threshold can be set" but do not specify a numerical value [S1].
*   **Sampling Interval:** Base interval for resampling. Increased when invalid points are encountered, restored when valid points are found [S1].

### Edge Cases and Pitfalls
*   **Boundary Interpolation Errors:** Interpolating values at the edges of sub-data blocks can result in values greater than the maximum or smaller than the minimum of the original block. This is mitigated by expanding each sub-data block by one voxel during BVH construction [S1].
*   **Empty Node Overhead:** Unlike octrees, the BVH construction avoids recording many empty nodes, which reduces computational complexity [S1]. However, if the BVH is not constructed efficiently, traversal overhead may negate benefits [G].
*   **Dynamic Data:** The BVH can be dynamically updated if the dataset changes frequently, unlike some static structures [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Construction:** The method involves an octree pre-processing step and BVH construction. The complexity depends on the volume size and the subdivision strategy. The sources note that compared to octree-based methods, this BVH method avoids recording many empty nodes, which increases computational complexity in those alternative methods [S1].
*   **Rendering:** The time complexity per ray is reduced by skipping empty spaces and early termination. The adaptive sampling and grouped interpolation further reduce the cost of interpolation calculations [S1].

### Measured Performance
*   **Hardware:** AMD Ryzen 5 5600H with Radeon Graphics 3.30 GHz, Windows 10 64-bit, 16GB memory [S1].
*   **Software:** Visual Studio 2019, Unity3D, C#, HLSL [S1].
*   **Results:** The method reduces unnecessary space debris, lowering hardware load and improving rendering speed, meeting real-time rendering requirements for large-scale aerosols [S2]. It significantly shortens total interpolation calculation time compared to traditional algorithms [S1].

### Strengths and Limitations
*   **Strengths:**
    *   Avoids recording many empty nodes compared to octree-based methods [S1].
    *   Not heavily dependent on the spatial position of non-empty voxels [S1].
    *   Can be dynamically updated if the dataset changes frequently [S1].
    *   Reduces hardware load and improves rendering speed for large-scale aerosols [S2].
*   **Limitations:**
    *   The specific numerical values for parameters (like $T_{term}$ and interpolation level ranges) are not provided, requiring empirical tuning [S1].
    *   The method is tailored for large-scale aerosols; performance on other types of volumetric data is not explicitly detailed [S1, S2].

## Background

### Origin and History
The method was proposed in the paper "Large-scale Aerosols Visualization Based on the Accelerated Ray Casting Algorithm" [S1]. It was developed to address the disadvantages of traditional ray casting algorithms, such as low efficiency, slow speed, and large resource consumption when rendering aerosol data (smoke, fog, dust) in 3D scenes [S1, S2].

### Development
The method combines several optimization techniques:
1.  **BVH for Empty-Space Skipping:** Uses a hierarchical bounding box tree to cull empty voxels [S1].
2.  **Early Termination:** Stops compositing when cumulative opacity exceeds a threshold [S1].
3.  **Adaptive Sampling:** Adjusts sampling intervals based on data validity [S1].
4.  **Grouped Linear Interpolation:** Reduces interpolation cost by varying complexity based on distance from the viewpoint [S1].

### Usage
The method is used for visualizing large-scale aerosols, including smoke, fog, and dust, in 3D scenes [S1, S2]. It is implemented using C# and HLSL in Unity3D [S1].

## Sources

[S1] Large-scale Aerosols Visualization Based on the Accelerated Ray Casting Algorithm, https://doi.org/10.54097/fcis.v1i2.1722, fetched 2026-10-24
[S2] (PDF) A Comparison between GPU-based Volume Ray Casting ..., https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
