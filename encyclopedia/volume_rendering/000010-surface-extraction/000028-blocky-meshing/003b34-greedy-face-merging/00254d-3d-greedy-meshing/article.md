# 3D Greedy Meshing


## Overview
3D Greedy Meshing extends 2D greedy meshing to three dimensions by extracting 2D slices from a 3D voxel grid along each axis and processing them with the 2D algorithm to generate a reduced set of faces representing the external surface [S1, unofficial]. It solves the problem of high triangle counts in voxel-based rendering by merging adjacent visible faces of the same voxel type into larger quads, achieving a simplified mesh that has at most 8x the number of faces that a perfect mesh would have [S1, unofficial]. It differs from its siblings by explicitly decomposing the 3D problem into six directional 2D sub-problems rather than operating directly on 3D volumes or using run-length encoding along axes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `voxel_grid` of type `GRID_3D` (`list[list[list[int]]]`), where each integer represents a voxel type [S1, unofficial].
- Output: `faces`, a `list[Face]` [S1, unofficial].

**Data Structures:**
- `Face`: A dataclass with fields:
  - `position`: `tuple[int, int, int]` representing the corner of the face [S1, unofficial].
  - `dimensions`: `tuple[int, int]` representing the width and height of the face [S1, unofficial].
  - `orientation`: `str` indicating the face normal direction ('top', 'bottom', 'front', 'back', 'left', 'right') [S1, unofficial].
  - `type`: `int` representing the voxel type of the face [S1, unofficial].

**Algorithm Steps:**
1. Define a helper function `is_exposed(x, y, z)` which returns `True` if the voxel at `(x, y, z)` is outside the grid bounds ($0 \le x < \text{GRID\_SIZE}$, etc.) or if `voxel_grid[z][y][x] == 0`. Otherwise, return `False` [S1, unofficial].
2. Define a helper function `get_face(grid, axis, index, offset)` which extracts a 2D slice. It iterates $y$ and $x$ over `GRID_SIZE`. For each axis ('z', 'y', 'x'), it checks if the voxel at the offset position is exposed using `is_exposed`. If exposed, it appends the voxel value; otherwise, it appends 0. It returns the 2D face grid and the orientation string from `NAME_MAPPING` [S1, unofficial].
3. Initialize an empty list `faces`. Iterate `i` from 0 to `GRID_SIZE`-1. For each `i`, iterate `offset` in `+1, -1`. For each `offset`, iterate `axis` in `['z', 'y', 'x']` [S1, unofficial].
4. Call `get_face(voxel_grid, axis, i, offset)` to get the 2D slice and orientation. Pass the 2D slice to `greedy_meshing_2d` to get a list of 2D meshes [S1, unofficial].
5. For each 2D mesh, construct a 3D `Face`. The position is $(x, y, i)$ if axis is 'z', $(i, x, y)$ if axis is 'x', or $(x, i, y)$ if axis is 'y'. The dimensions, orientation, and type are taken from the 2D mesh. Append to `faces` [S1, unofficial].

**Parameters:**
- `GRID_SIZE`: The dimension of the cubic voxel grid. In the example, `GRID_SIZE = 3` [S1, unofficial].
- `NAME_MAPPING`: A dictionary mapping axis and offset tuples to orientation strings. ('z', +1): 'top', ('z', -1): 'bottom', ('y', +1): 'front', ('y', -1): 'back', ('x', +1): 'right', ('x', -1): 'left' [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from dataclasses import dataclass
from typing import List, Tuple

@dataclass
class Face:
    position: Tuple[int, int, int]
    dimensions: Tuple[int, int]
    orientation: str
    type: int

GRID_VALUE_TYPE = int
GRID_2D = List[List[GRID_VALUE_TYPE]]
GRID_3D = List[List[List[GRID_VALUE_TYPE]]]

NAME_MAPPING = {
    ('z', 1): 'top',
    ('z', -1): 'bottom',
    ('y', 1): 'front',
    ('y', -1): 'back',
    ('x', 1): 'right',
    ('x', -1): 'left'
}

def is_exposed(grid: GRID_3D, x: int, y: int, z: int) -> bool:
    grid_size = len(grid)
    if not (0 <= x < grid_size and 0 <= y < grid_size and 0 <= z < grid_size):
        return True
    return grid[z][y][x] == 0

def get_face(grid: GRID_3D, axis: str, index: int, offset: int) -> Tuple[GRID_2D, str]:
    grid_size = len(grid)
    face_grid = [[0] * grid_size for _ in range(grid_size)]
    orientation = NAME_MAPPING[(axis, offset)]
    
    for y in range(grid_size):
        for x in range(grid_size):
            # Determine neighbor coordinate in the offset direction
            nx, ny, nz = x, y, index
            if axis == 'x':
                nx = index + offset
            elif axis == 'y':
                ny = index + offset
            elif axis == 'z':
                nz = index + offset
                
            if is_exposed(grid, nx, ny, nz):
                # Store the voxel type at the slice plane
                if axis == 'x':
                    face_grid[y][x] = grid[z][y][index]
                elif axis == 'y':
                    face_grid[y][x] = grid[z][index][x]
                elif axis == 'z':
                    face_grid[y][x] = grid[index][y][x]
            else:
                face_grid[y][x] = 0
    return face_grid, orientation

def greedy_meshing_3d(voxel_grid: GRID_3D) -> List[Face]:
    grid_size = len(voxel_grid)
    faces = []
    axes = ['z', 'y', 'x']
    offsets = [1, -1]
    
    for i in range(grid_size):
        for offset in offsets:
            for axis in axes:
                slice_grid, orientation = get_face(voxel_grid, axis, i, offset)
                # Subroutine to merge 2D faces into quads
                two_d_faces = greedy_meshing_2d(slice_grid) 
                
                for face_2d in two_d_faces:
                    u, v = face_2d.position
                    w, h = face_2d.dimensions
                    ftype = face_2d.type
                    
                    # Map 2D coordinates back to 3D based on axis
                    if axis == 'z':
                        pos = (u, v, i)
                    elif axis == 'x':
                        pos = (i, u, v)
                    elif axis == 'y':
                        pos = (u, i, v)
                    
                    faces.append(Face(position=pos, dimensions=(w, h), orientation=orientation, type=ftype))
    return faces
```

**Edge Cases and Pitfalls:**
- Boundaries: The `is_exposed` function handles grid boundaries by returning `True`, ensuring faces on the exterior of the grid are generated [S1, unofficial].

## Subtopics

### [Voxel Mesh Generation (Run-based Greedy Meshing)](3127-voxel-mesh-generation-run-based-greedy-meshing.md)
Uses run-length encoding along axes to merge faces, potentially faster for large uniform regions.

### [Chunked Voxel Mesh Generation with Face Culling and Quad Expansion](3133-chunked-voxel-mesh-generation-with-face-culling-and-quad-exp.md)
Processes voxels in chunks to improve cache locality and allows for more complex culling strategies.

### [Binary Greedy Meshing with Packed Quads](3312-binary-greedy-meshing-with-packed-quads.md)
Optimizes memory usage by packing quad data into binary formats, suitable for GPU transfer.

### [Greedy Meshing for Voxel Faces](3367-greedy-meshing-for-voxel-faces.md)
Generalizes the greedy approach to non-cubic voxel shapes.

## Operating characteristics
- Measured Performance: Not given in the sources.

## Background
Origin: The method is described in an unofficial GitHub repository by Bertil Braun [S1, unofficial]. It extends the 2D greedy meshing algorithm to 3D [S1, unofficial].
Usage: Used in computer graphics, games, and simulations to improve performance by reducing the number of faces [S1, unofficial].

## Sources
[S1] GitHub - BertilBraun/Greedy-Meshing-in-Python: A simple 2D and 3D ..., https://github.com/BertilBraun/Greedy-Meshing-in-Python, fetched 2026-10-27 (unofficial)
