# Greedy-Meshing-Reloaded
In this repository I will refactor and upgrade my Greedy Meshing algorithm from [Voxels](https://github.com/gilchristb78/Voxel).

## Generate Mesh
this function does the bulk of our greedy Meshing, figuring out which faces to render, and which faces to combine before rendering. <br><br>
To Start we loop through the 3 rotations of a plane that coorespond to our 3 axis. <br>
![3 Axis Photo](https://github.com/gilchristb78/Greedy-Meshing-Reloaded/blob/main/GreedyMeshCapture/CaptureGreedyAxis.PNG)<br>
Then on Each Axis we loop through all the positions that the plane could touch the side of a block. From before the first block to after the last block. <br>
![X Axis Plane Traversing](https://github.com/gilchristb78/Greedy-Meshing-Reloaded/blob/main/GreedyMeshCapture/CaptureGreedyAxisX.PNG)<br>
![Y Axis Plane Traversing](https://github.com/gilchristb78/Greedy-Meshing-Reloaded/blob/main/GreedyMeshCapture/CaptureGreedyAxisY.PNG)<br>
![Z Axis Plane Traversing](https://github.com/gilchristb78/Greedy-Meshing-Reloaded/blob/main/GreedyMeshCapture/CaptureGreedyAxisZ.PNG)<br>
Next we loop through each position that the plane touches and check the two corresponding blocks. <br>
![Z Axis Plane Block Checking](https://github.com/gilchristb78/Greedy-Meshing-Reloaded/blob/main/GreedyMeshCapture/CaptureGreedyAxisZChecks.PNG) <br>
If both blocks are solid or both blocks are air then we do not need to spawn a face and thus set the block to null and normal to 0. <br>
However if only one block is air and the other is solid we then know a block should spawn and set block to the corresponding block and normal to 1 or -1 depending on which side the solid block is relative to the plane. <br>
This results in a mask with the following indices.<br>
![Mask Indices](https://github.com/gilchristb78/Greedy-Meshing-Reloaded/blob/main/GreedyMeshCapture/CaptureGreedyMask.PNG) <br>
And the following Values. Color = Block Type (gray is stone, purple is Null) and Number = Normal.<br>
![Mask Values](https://github.com/gilchristb78/Greedy-Meshing-Reloaded/blob/main/GreedyMeshCapture/CaptureGreedyMaskValues.PNG) <br>
Then we take the mask weve created and perform that part of greedy meshing that combines like faces. <br>
![Greedy Meshing Animation](https://github.com/gilchristb78/Greedy-Meshing-Reloaded/blob/main/GreedyMeshCapture/greedy_anim.svg) <br>
once we finish combining faces to get a single combined "face" we then pass that data on to create quad which converts this data into somehting that can be used by UE5 to create our mesh. <br>


## Goals

- [ ] Refactor / Cleanup the code
- [X] Understand and document how the greedy mesh works
- [X] Block Size
- [X] Block Textures / Materials
  - [X] All Stone Textures
  - [X] Dirt / Grass
  - [X] Sand
  - [X] Water (translucent)
    - [X] New Mesh Section
- [X] Fix border of chunks always renders the faces
  - [X] Chunk.GetBlock (if x or y less than 0 or greater than size, call GetOwner().getBlock)
  - [X] Change rendering so it doesnt auto render at spawn it waits till DrawDist square is populated with blocks then call render
  - [ ] Newly Rendered Chunks call Render on 4 adjacent chunks to despawn those edges.
- [ ] New Block Structure
  - [ ] Which sides are opaque (to be used by greedy mesh)
  - [ ] Rotation
- [ ] New Shapes
  - [ ] Slopes
  - [ ] Corners (not Greedy | could be greedy with slopes nearby but probly not worth it)
  - [ ] Inside Corners (same as above)
  - [ ] New Shapes slope off surface
- [ ] Caves
  - [ ] Basic
  - [ ] Perlin Caves
  - [ ] With New Smoothing Shapes
    - [ ] Ground
    - [ ] Ceiling
- [ ] Biomes
  - [ ] Voronoi noise
  - [ ] Block Types
    - [ ] Colors
    - [ ] Types
  - [ ] Terrain Generation
  - [ ] Structures (later)
- [ ] Additions
  - [ ] Ground Cover (trees / Bushes)
  - [ ] Structures (castles / towns / etc)
