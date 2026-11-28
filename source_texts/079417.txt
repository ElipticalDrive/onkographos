## Voxelize an AutoCAD solid

To load:
﻿netload "Voxelize\bin\x64\Debug\net8.0-windows\Voxelize.dll"

 To use:
 Command is "VoxelizeSolid"
 It will ask you to pick an AutoCAD solid, then ask you for a suggested voxel size (length of one edge).

 The suggested voxel size is only a suggestion.
 This suggestion is used to create the actual voxel edge length.
 The suggestion is adjusted so that the voxels evenly fill the longest axis of the solid3d extents.
