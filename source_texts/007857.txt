# Fast Voxel Traversal Algorithm  
#### A Fast Voxel Traversal Algorithm implemention for Unity3D.
-----
Forked from https://github.com/Victorique-GOSICK/Voxel-3D-DDA.  
The original owner named it as "**Voxel 3D DDA**" but actually it is "**Fast Voxel Traversal Algorithm**" from **John Amanatides** and **Andrew Woo**.  
May be he/she firstly implemented **the DDA** but changed to **the FVTA** instead.

The codes from GOSICK has a problem: when the Ray origin from outside of the grid, he/she snapped the "out of the box" cell into inside the grid.  
That made the algorithm become wrongly until the ray really intersect the grid. May be that is his/her intention for his/her work.  

So, I updated the code a bit to make it corrected generally.
