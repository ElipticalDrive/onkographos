# 3D-Point-cloud-filter
ROS 2 node for filtering and downsampling 3D LiDAR point clouds with field-of-view, range, and voxel grid constraints.

## Features

- Field-of-view filtering (±45° horizontal, ±30° vertical)
- Range and height clipping
- Voxel grid downsampling
- TF-aware transformation and synchronized timestamping
- Republish original and filtered clouds, and 2D scan

## Dependencies

- ROS 2 (Humble or newer)
- PCL
- `tf2_ros`, `sensor_msgs`, `pcl_conversions`, `tf2_sensor_msgs`

## Build

```bash
cd ~/ros2_ws/src
git clone https://github.com/your_username/ouster_cloud_filter.git
cd ~/ros2_ws
colcon build --packages-select ouster_cloud_filter
source install/setup.bash
