# High Resolution Sparse Voxel DAGs
This implementation is based on [Github page](https://github.com/Forceflow/ooc_svo_builder) library for handling octree IO operations.

My implementation files of SVDAG constructor from SVO are in the folder src/svdag_construction:
* SVDAG_main.cpp
* SVDAG.h
* SVDAG.cpp
* SVDAG_node.cpp
* SVDAG_node.h

## Dependencies

The libraries needed are included in src/libs:
* **libmorton**
* **libtri**
* **trimesh2**
* **glm**

They are already set up and were built for Linux_x64 system. You could rebuild it using cmake

## Building 

After the libraries are build, the building scripts should be executed.

To rebuild the **tri_convert**, **svo_constructor** and **svdag_constructor** executables in the 'building_scripts' folder:
* ./build_tri_convert.sh
* ./build_svo.sh
* ./build_svdag.sh

If all the building scripts were executed successfully, the system is set up and could be used.

## Execution pipeline

* **tri_convert:** A tool to convert any model file to a simple, streamable .tri format.
* **svo_constructor:** Out-Of-Core SVO Constructor: Partitioning, voxelizing and SVO Building rolled into one executable, needs a .tri file as input and return .octree, .octreenodes, .octreedata files as output.
* **svdag_constructor:** My implementation of the DAG reducing algorithm, needs the previous .octree, .octreenodes, .octreedata files and return a .dag file as output.

Execution steps:
**Step 1**: Create .tri file from .ply file 
`./tri_convert -f ../models/bunny.ply`

**Step 2**: Convert tri to octree file
`./svo_constructor -f ../models/bunny.tri`

**Step 3**: Create DAG from Octree
`./svdag_constructor -f ../models/bunny1024_1.octree`
