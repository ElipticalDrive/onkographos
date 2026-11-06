# VoxelFilter
`class VoxelFilter` is based on PCL's `class VoxelGrid`.

PCL's `VoxelGrid` cannot inherit data besides position(x, y, z);

In PCL's `VoxelGrid`, create output points by: 
```C++
output.points.resize (total);
```
This line cannot save non-position data from input point cloud.

And only set position by: 
```C++
output.points[index].getVector4fMap () = centroid;
```

Our `Voxel Filter` update this feature and make it friendly for custom point.
