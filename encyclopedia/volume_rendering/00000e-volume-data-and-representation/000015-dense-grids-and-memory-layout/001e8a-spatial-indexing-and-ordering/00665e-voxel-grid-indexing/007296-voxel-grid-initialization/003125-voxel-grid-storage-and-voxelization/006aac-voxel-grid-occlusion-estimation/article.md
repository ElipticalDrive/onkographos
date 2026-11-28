# Voxel Grid Occlusion Estimation


## Overview
Voxel Grid Occlusion Estimation is a method to estimate occluded space in a scene by building a voxel grid from a point cloud and performing ray traversal from a sensor origin to target voxels to determine if they are free (0) or occluded/occupied (1) [S1]. It differs from standard voxelization by adding a visibility check via ray casting, distinguishing it from [VoxelizationGrid](6601-voxelizationgrid.md) which only computes occupancy, and from [Instanced Voxel Rendering](10592-instanced-voxel-rendering.md) which focuses on rendering rather than geometric occlusion analysis [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A point cloud (`PointCloud`), sensor origin (`sensor_origin_`), sensor orientation (`sensor_orientation_`), and leaf size (`leaf_size_`).
- **Output:** A set of voxel coordinates (`std::vector<Eigen::Vector3i>`) representing occluded voxels, or the state (0 or 1) of a specific target voxel.

**Data Structures:**
The class inherits from `VoxelGrid<PointT>` and uses the following fields [S1]:
- `initialized_` (bool): Flag indicating if the grid has been built.
- `leaf_size_` (Eigen::Vector4f): Size of voxel cells.
- `inverse_leaf_size_` (Eigen::Array4f): Inverse of leaf size for coordinate conversion.
- `b_min_`, `b_max_` (Eigen::Vector4f): Minimum and maximum bounding box coordinates.
- `sensor_origin_` (Eigen::Vector4f): Position of the sensor.
- `sensor_orientation_` (Eigen::Quaternionf): Orientation of the sensor.
- `filtered_cloud_` (PointCloud): The voxelized point cloud data.

**Steps:**

1.  **Initialization:**
    The constructor sets `initialized_` to `false` and calls `setSaveLeafLayout(true)` to ensure leaf layout information is saved [S1]. The user must call `initializeVoxelGrid()` to build the voxel grid from the input cloud and compute additional values for ray traversal [S1].

2.  **Coordinate Conversion (World to Grid):**
    To map a world point $(x, y, z)$ to grid indices $(i, j, k)$, round the scaled coordinates [S1]:
    $$
    i = \text{round}(x \cdot \text{inverse\_leaf\_size\_}[0])
    $$
    $$
    j = \text{round}(y \cdot \text{inverse\_leaf\_size\_}[1])
    $$
    $$
    k = \text{round}(z \cdot \text{inverse\_leaf\_size\_}[2])
    $$
    Where $\text{round}(d) = \lfloor d + 0.5 \rfloor$ [S1].

3.  **Coordinate Conversion (Grid to World):**
    To get the centroid $(x, y, z)$ of a voxel $(i, j, k)$, first adjust indices for negative bounds [S1]:
    $$
    i' = \begin{cases} |b_{min}[0]| + i & \text{if } b_{min}[0] < 0 \\ i - b_{min}[0] & \text{otherwise} \end{cases}
    $$
    (Similarly for $j'$ and $k'$) [S1]. Then compute the centroid [S1]:
    $$
    x = b_{min}[0] + (\text{leaf\_size}[0] \cdot 0.5) + (i' \cdot \text{leaf\_size}[0])
    $$
    $$
    y = b_{min}[1] + (\text{leaf\_size}[1] \cdot 0.5) + (j' \cdot \text{leaf\_size}[1])
    $$
    $$
    z = b_{min}[2] + (\text{leaf\_size}[2] \cdot 0.5) + (k' \cdot \text{leaf\_size}[2])
    $$

4.  **Ray-Box Intersection:**
    Compute $t_{min}$, the scaling value where the ray intersects the voxel grid bounding box, using `rayBoxIntersection(origin, direction)` [S1]. The entry point is $p_{entry} = origin + t_{min} \cdot direction$ [S1].

5.  **Ray Traversal:**
    Determine the state of a target voxel $(i, j, k)$ by traversing from the sensor origin to the target. The function `rayTraversal` returns 0 if visible/free and 1 if occupied/occluded. It uses the Amanatides and Woo algorithm [S1].

6.  **Batch Estimation:**
    `occlusionEstimationAll` computes the coordinates of all occluded voxels in the grid by iterating through the grid and calling the ray traversal for each voxel (or subset thereof) to determine if it is occluded from the sensor origin [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelGridOcclusionEstimation:
    def __init__(self):
        self.initialized_ = False
        self.setSaveLeafLayout(True)
        self.sensor_origin_ = Vector4f(0, 0, 0, 0)
        self.sensor_orientation_ = Quaternionf(1, 0, 0, 0)
        self.b_min_ = Vector4f(0, 0, 0, 0)
        self.b_max_ = Vector4f(0, 0, 0, 0)
        self.leaf_size_ = Vector4f(0, 0, 0, 0)
        self.inverse_leaf_size_ = Array4f(0, 0, 0, 0)
        self.filtered_cloud_ = PointCloud()

    def setLeafSize(self, leaf_x, leaf_y, leaf_z):
        self.leaf_size_[0] = leaf_x
        self.leaf_size_[1] = leaf_y
        self.leaf_size_[2] = leaf_z
        self.inverse_leaf_size_[0] = 1.0 / leaf_x
        self.inverse_leaf_size_[1] = 1.0 / leaf_y
        self.inverse_leaf_size_[2] = 1.0 / leaf_z

    def initializeVoxelGrid(self):
        # Build voxel grid from input cloud and compute additional values
        # for ray traversal algorithm [S1]
        self.initialized_ = True

    def getGridCoordinatesRound(self, x, y, z):
        return Vector3i(
            int(floor(x * self.inverse_leaf_size_[0] + 0.5)),
            int(floor(y * self.inverse_leaf_size_[1] + 0.5)),
            int(floor(z * self.inverse_leaf_size_[2] + 0.5))
        )

    def getCentroidCoordinate(self, ijk):
        i = ijk[0]
        j = ijk[1]
        k = ijk[2]
        
        # Adjust indices for negative bounds [S1]
        if self.b_min_[0] < 0:
            i = abs(self.b_min_[0]) + i
        else:
            i = i - self.b_min_[0]
            
        if self.b_min_[1] < 0:
            j = abs(self.b_min_[1]) + j
        else:
            j = j - self.b_min_[1]
            
        if self.b_min_[2] < 0:
            k = abs(self.b_min_[2]) + k
        else:
            k = k - self.b_min_[2]
            
        xyz = Vector4f(
            self.b_min_[0] + (self.leaf_size_[0] * 0.5) + (float(i) * self.leaf_size_[0]),
            self.b_min_[1] + (self.leaf_size_[1] * 0.5) + (float(j) * self.leaf_size_[1]),
            self.b_min_[2] + (self.leaf_size_[2] * 0.5) + (float(k) * self.leaf_size_[2]),
            0
        )
        return xyz

    def rayBoxIntersection(self, origin, direction):
        # Compute t_min where ray intersects bounding box [S1]
        # Implementation details of Amanatides and Woo ray-box intersection
        # Not fully specified in source, assumed to return float t_min
        pass

    def rayTraversal(self, target_voxel, origin, direction, t_min):
        # Traverse from origin to target_voxel using Amanatides and Woo algorithm [S1]
        # Returns 0 if visible/free, 1 if occupied/occluded
        pass

    def occlusionEstimation(self, target_voxel):
        if not self.initialized_:
            return -1
        
        # Get sensor direction from orientation
        direction = self.sensor_orientation_ * Vector4f(0, 0, 1, 0) # Assuming forward axis
        
        t_min = self.rayBoxIntersection(self.sensor_origin_, direction)
        if t_min < 0:
            return -1 # Ray does not intersect bounding box
            
        state = self.rayTraversal(target_voxel, self.sensor_origin_, direction, t_min)
        return state

    def occlusionEstimationAll(self):
        occluded_voxels = []
        # Iterate through all voxels in the grid
        # This part is not fully detailed in the source, but implies iterating
        # through the grid structure built in initializeVoxelGrid
        for voxel_index in self.get_all_voxel_indices():
            state = self.occlusionEstimation(voxel_index)
            if state == 1:
                occluded_voxels.append(voxel_index)
        return occluded_voxels
```

**Parameters:**
- **Leaf Size:** The size of the voxel grid cells, stored in `leaf_size_` and `inverse_leaf_size_`. Set via `setLeafSize` [S1].
- **Sensor Origin:** The position of the sensor, stored in `sensor_origin_` [S1].
- **Sensor Orientation:** The orientation of the sensor, stored in `sensor_orientation_` [S1].

**Edge Cases and Pitfalls:**
- The method requires `initializeVoxelGrid()` to be called before any estimation [S1].
- Negative bounding box minimums require specific index adjustment logic to correctly map grid coordinates to world coordinates [S1].
- Precision issues may arise in `round` operations if coordinates are very close to voxel boundaries; the source uses `floor(d + 0.5)` which is standard but may behave differently than other rounding modes [S1].

## Operating characteristics
Not given in the sources.

## Background
The method is implemented in the Point Cloud Library (PCL) by Christian Potthast [S1]. It is based on the ray traversal algorithm described by John Amanatides and Andrew Woo in "A Fast Voxel Traversal Algorithm for Ray Tracing" [S1].

## Sources
[S1] Point Cloud Library (PCL): pcl/filters/voxel_grid_occlusion_estimation.h Source File, https://pointclouds.org/documentation/voxel__grid__occlusion__estimation_8h_source.html, fetched 2026-10-24
