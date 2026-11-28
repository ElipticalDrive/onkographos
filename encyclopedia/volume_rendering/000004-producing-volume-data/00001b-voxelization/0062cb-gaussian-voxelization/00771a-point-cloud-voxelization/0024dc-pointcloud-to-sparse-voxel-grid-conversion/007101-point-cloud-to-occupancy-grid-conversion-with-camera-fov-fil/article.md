# Point Cloud to Occupancy Grid Conversion with Camera FOV Filtering

## Overview
This method converts a LiDAR point cloud into a 3D occupancy grid by filtering points based on the camera's Field of View (FOV) and projecting them into voxel coordinates [S1, unofficial]. It differs from general voxelization by strictly limiting the input points to those visible within the camera's intrinsic and extrinsic parameters, ensuring the resulting grid aligns with the visual sensor's perspective [S1, unofficial].

## How it works
**Inputs:**
- `velodyine_points`: A 4xN array of homogeneous LiDAR coordinates $(X_v, Y_v, Z_v, 1)$ [S1, unofficial].
- `R`: 3x3 rotation matrix from LiDAR to camera frame [S1, unofficial].
- `T`: 3x1 translation vector from LiDAR to camera frame [S1, unofficial].
- `P_rect`: 3x3 projection matrix derived from camera calibration [S1, unofficial].
- `roi_02`: Region of Interest bounds $(x, y, w, h)$ for 2D validation [S1, unofficial].
- `v_fov`: Vertical FOV range, typically $(-24.9^\circ, 2.0^\circ)$ [S1, unofficial].
- `h_fov`: Horizontal FOV range, typically $(-45^\circ, 45^\circ)$ [S1, unofficial].
- `self.occupancy_shape`: Tuple $(H, W, D)$ defining the grid dimensions [S1, unofficial].
- `self.occupancy_mask_2d_shape`: Tuple $(H, W)$ for the 2D mask [S1, unofficial].
- `self.grid_x`, `self.grid_y`, `self.grid_z`: Grid resolution parameters [S1, unofficial].
- `self.occ_x`, `self.occ_y`, `self.occ_z`: Occupancy scaling parameters [S1, unofficial].

**Outputs:**
- `occupancy_grid`: 3D array of shape `self.occupancy_shape` with dtype `float32`, where occupied voxels are set to 1.0 [S1, unofficial].
- `occupancy_mask_2d`: 2D array of shape `self.occupancy_mask_2d_shape` with dtype `uint8`, storing depth-derived intensity [S1, unofficial].
- `velodyine_points_camera`: Array of filtered points in camera coordinates [S1, unofficial].

**Data Structures:**
- `RT_`: 3x4 transformation matrix formed by concatenating `R` and `T` along axis 1 [S1, unofficial].
- `image_points`: 3xN temporary array for pixel coordinates [S1, unofficial].

**Steps:**
1. **FOV Filtering:** Filter `velodyine_points` using `velo_points_filter` with `v_fov` and `h_fov` [S1, unofficial].
2. **Initialization:** Create `occupancy_grid` as zeros of shape `self.occupancy_shape` and dtype `float32`. Create `occupancy_mask_2d` as zeros of shape `self.occupancy_mask_2d_shape` and dtype `uint8` [S1, unofficial].
3. **Coordinate Transformation:** Compute `RT_` by concatenating `R` and `T` [S1, unofficial]. Transform points from Velodyne to camera coordinates by multiplying `RT_` with each point column [S1, unofficial]. Remove the 4th row (homogeneous coordinate) from the point array [S1, unofficial].
4. **Projection:** Multiply `P_rect` with each camera coordinate column to get homogeneous image points [S1, unofficial]. Normalize by dividing the first two rows by the third row (Z-depth) [S1, unofficial]. Remove the 3rd row (Z-depth) to obtain 2D pixel coordinates $(x, y)$ [S1, unofficial].
5. **Axis Permutation:** Apply coordinate transformation: $x_{new} = z$, $y_{new} = x$, $z_{new} = -y$ [S1, unofficial].
6. **Grid Index Calculation:** Calculate voxel indices $(i, j, k)$ using:
   $$ i = \left\lfloor \frac{\lfloor x \cdot \frac{self.occ\_x}{2} \rfloor}{self.grid\_x} \right\rfloor \cdot 2 $$
   $$ j = \left\lfloor \frac{y \cdot \frac{self.occ\_y}{2}}{self.grid\_y} + \frac{self.occ\_y}{2} \right\rfloor $$
   $$ k = \left\lfloor \frac{z \cdot \frac{self.occ\_z}{2}}{self.grid\_z} + \frac{self.occ\_z}{2} \right\rfloor $$
   [S1, unofficial].
7. **Validation:** Check if the point is within the camera ROI ($0 \le img\_x < w$ and $0 \le img\_y < h$) AND within the occupancy grid bounds ($0 < i < self.occupancy\_shape[0]$, $0 < j < self.occupancy\_shape[1]$, $0 < k < self.occupancy\_shape[2]$) [S1, unofficial].
8. **Update:** If valid, append the camera coordinate point to `velodyine_points_camera` and set `occupancy_grid[i,j,k] = 1.0` [S1, unofficial].
9. **Mask Update:** Update `occupancy_mask_2d[i,j]` with the maximum of its current value and the calculated depth intensity:
   $$ \text{val} = \max\left(0, \frac{k-6}{15-6}\right) $$
   $$ \text{mask\_val} = \min(255, 255 \cdot \text{val}) $$
   $$ occupancy\_mask\_2d[i,j] = \max(\text{mask\_val}, occupancy\_mask\_2d[i,j]) $$
   [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def transform_points_to_occupancy_grid(velodyine_points, R, T, P_rect, roi_02, v_fov, h_fov, 
                                       occupancy_shape, occupancy_mask_2d_shape, 
                                       grid_x, grid_y, grid_z, occ_x, occ_y, occ_z):
    # 1. FOV Filtering
    # Note: velo_points_filter implementation is external; assume it returns filtered points
    velodyine_points = velo_points_filter(velodyine_points, v_fov, h_fov)
    
    # 2. Initialization
    occupancy_grid = np.zeros(occupancy_shape, dtype=np.float32)
    occupancy_mask_2d = np.zeros(occupancy_mask_2d_shape, dtype=np.uint8)
    velodyine_points_camera = []
    
    x_roi, y_roi, w_roi, h_roi = roi_02
    
    # 3. Coordinate Transformation
    RT_ = np.concatenate((R, T), axis=1)
    velodyine_points_orig = velodyine_points.copy()
    
    # Transform to camera coordinates
    # Assuming velodyine_points is 4xN
    velodyine_points[:3, :] = np.matmul(RT_, velodyine_points)
    velodyine_points = np.delete(velodyine_points, 3, axis=0)
    
    # 4. Projection
    image_points = velodyine_points.copy()
    image_points = np.matmul(P_rect, image_points)
    
    # Normalize by Z-depth (3rd row)
    # Avoid division by zero
    z_depth = image_points[2, :]
    valid_z = z_depth != 0
    image_points[0, :] = np.divide(image_points[0, :], z_depth, out=np.zeros_like(image_points[0, :]), where=valid_z)
    image_points[1, :] = np.divide(image_points[1, :], z_depth, out=np.zeros_like(image_points[1, :]), where=valid_z)
    
    # Remove Z-depth row
    image_points = np.delete(image_points, 2, axis=0)
    
    # 5. Axis Permutation and Grid Calculation
    for index in range(image_points.shape[1]):
        img_x = image_points[0, index]
        img_y = image_points[1, index]
        
        # Get original camera coordinates before permutation for grid calc? 
        # The source uses velodyine_points[:, index] which has been permuted in previous steps?
        # Actually, the source code does: x, y, z = velodyine_points[:, index]
        # But velodyine_points was modified in step 3. 
        # The source code comment says "x, y, z = velodyine_points[:, index]"
        # And then "x, y, z = z, x, -y". 
        # So we use the current velodyine_points (camera coords)
        x_cam = velodyine_points[0, index]
        y_cam = velodyine_points[1, index]
        z_cam = velodyine_points[2, index]
        
        # Permutation
        x, y, z = z_cam, x_cam, -y_cam
        
        # Grid Indices
        i = int((x * (occ_x // 2)) // grid_x) * 2
        j = int((y * (occ_y // 2)) // grid_y + (occ_y // 2))
        k = int((z * (occ_z // 2)) // grid_z + (occ_z // 2))
        
        # 6. Validation
        if (0 <= img_x < w_roi and 0 <= img_y < h_roi) and \
           (0 < i < occupancy_shape[0] and 0 < j < occupancy_shape[1] and 0 < k < occupancy_shape[2]):
            
            velodyine_points_camera.append((x, y, z))
            occupancy_grid[i, j, k] = 1.0
            
            # Mask Update
            val = max(0, (k - 6) / (15 - 6))
            mask_val = min(255, 255 * val)
            occupancy_mask_2d[i, j] = max(int(mask_val), occupancy_mask_2d[i, j])
            
    velodyine_points_camera = np.array(velodyine_points_camera, dtype=np.float32)
    
    return {
        'occupancy_grid': occupancy_grid,
        'occupancy_mask_2d': occupancy_mask_2d,
        'velodyine_points_camera': velodyine_points_camera
    }
```

**Parameters:**
- `v_fov`: Vertical Field of View range, typically $(-24.9, 2.0)$ [S1, unofficial].
- `h_fov`: Horizontal Field of View range, typically $(-45, 45)$ [S1, unofficial].
- `P_rect`: 3x3 projection matrix derived from `self.calib_cam_to_cam['P_rect_02']` reshaped to (3,4) and sliced `[:3,:3]` [S1, unofficial].
- `RT_`: 3x4 transformation matrix concatenating rotation `self.R` and translation `self.T` along axis 1 [S1, unofficial].
- `roi_02`: Region of Interest bounds $(x, y, w, h)$ used for 2D image coordinate validation [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero in normalization if $Z$-depth is 0 [G].
- Integer division behavior in Python 3 vs 2; the source uses `//` which is floor division [S1, unofficial].
- The grid index calculation uses specific scaling factors (`occ_x`, `grid_x`) that are not defined in the source beyond their usage [S1, unofficial].
- The mask update formula assumes a specific depth range $(6, 15)$ for normalization [S1, unofficial].

## Operating characteristics
**Time Complexity:** Linear with respect to the number of points $N$ after filtering, $O(N)$, due to the single pass loop [S1, unofficial].
**Memory Complexity:** $O(H \cdot W \cdot D)$ for the occupancy grid and $O(H \cdot W)$ for the 2D mask, where $H, W, D$ are grid dimensions [S1, unofficial].
**Performance:** The method is designed to speed up conversion compared to general voxelization by filtering points early [S1, unofficial].
**Strengths:** Aligns voxel data with camera view; reduces noise from points outside the camera's view [S1, unofficial].
**Limitations:** Relies on specific calibration matrices and FOV ranges; not generalizable without these parameters [S1, unofficial].

## Background
This method originates from the KITTI dataset processing pipeline, specifically implemented in the `kitti_iterator` repository [S1, unofficial]. It addresses the need to efficiently convert Velodyne point clouds into occupancy grids for 3D object detection tasks, ensuring that only relevant points within the camera's field of view are processed [S1, unofficial].

## Sources
[S1] Faster conversion of Point Cloud to Occupancy Grid and vice versa, https://github.com/PW22-SBN-01/kitti_iterator/issues/1, fetched 2026-10-27 (unofficial)
