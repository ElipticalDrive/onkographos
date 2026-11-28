# Sparse Voxel Tree Data Layout

Also called: Tree class, bit-packed voxel grid

## Overview
This method defines the memory layout for the leaf-level occupancy and color data of a Sparse Voxel Octree (SVO). It solves the problem of efficiently storing sparse 3D voxel grids by packing 8 voxels along the x-axis into a single byte, reducing memory overhead compared to uncompressed grids. It differs from sibling methods like [GigaVoxels Data Synthesis](16702-gigavoxels-data-synthesis.md) by focusing on the raw bit-packing of the leaf nodes rather than hierarchical synthesis or mipmap generation.

## How it works
**Inputs:**
- Grid dimensions: `sx`, `sy`, `sz` (integers, typically powers of 2).
- Optional color support: `usecolor` (boolean).

**Outputs:**
- A contiguous memory block for occupancy bits.
- Two optional contiguous memory blocks for color channels (if `usecolor` is true).

**Data Structures:**
The layout uses three arrays of bytes:
1. `voxel`: Occupancy data. Size: `(sx / 8) * sy * sz` bytes.
2. `voxel_col1`: First color channel (e.g., Red). Size: Same as `voxel`.
3. `voxel_col2`: Second color channel (e.g., Green). Size: Same as `voxel`.

Each byte in these arrays stores 8 voxels along the x-axis. The bit index within the byte corresponds to the x-coordinate modulo 8.

**Step-by-Step Initialization:**
1. Calculate the stride `sxy = sx * sy`.
2. Calculate the total number of bytes `N = (sx / 8) * sy * sz`.
3. Allocate `voxel` array of size `N` and initialize all bytes to 0.
4. If `usecolor` is true:
   - Allocate `voxel_col1` array of size `N` and initialize to 0.
   - Allocate `voxel_col2` array of size `N` and initialize to 0.
5. If `usecolor` is false:
   - Set pointers for `voxel_col1` and `voxel_col2` to null.

**Access Formula:**
To access voxel at coordinates `(i, j, k)`:
- Byte index: `idx = (i + j * sx + k * sxy) >> 3` (equivalent to `(i + j * sx + k * sx * sy) / 8`).
- Bit mask: `mask = 1 << (i & 7)`.

**Setting a Voxel:**
To set voxel `(i, j, k)` to occupied:
`voxel[idx] |= mask`

**Edge Cases:**
- `sx` must be divisible by 8. If not, the layout is invalid or requires padding [S1, unofficial].
- Coordinates must be within bounds `[0, sx-1]`, `[0, sy-1]`, `[0, sz-1]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def init_svo_layout(sx, sy, sz, usecolor):
    # Validate dimensions
    if sx % 8 != 0:
        raise ValueError("sx must be divisible by 8")
    
    sxy = sx * sy
    num_bytes = (sx // 8) * sy * sz
    
    # Allocate and zero occupancy array
    voxel = [0] * num_bytes
    
    # Allocate color arrays if needed
    voxel_col1 = None
    voxel_col2 = None
    
    if usecolor:
        voxel_col1 = [0] * num_bytes
        voxel_col2 = [0] * num_bytes
        
    return {
        'voxel': voxel,
        'voxel_col1': voxel_col1,
        'voxel_col2': voxel_col2,
        'sx': sx,
        'sy': sy,
        'sz': sz,
        'sxy': sxy,
        'num_bytes': num_bytes
    }

def set_voxel(layout, i, j, k, occupied=True):
    sxy = layout['sxy']
    idx = (i + j * layout['sx'] + k * sxy) >> 3
    mask = 1 << (i & 7)
    
    if occupied:
        layout['voxel'][idx] |= mask
    else:
        layout['voxel'][idx] &= ~mask
        
    # Note: Color setting logic is not specified in sources [S1, unofficial]
    # Implementers must define how color bits are packed if usecolor is true.
```

## Subtopics

### [Voxel Generation: Sphere, Cube, Branch, Tree](21258-voxel-generation-sphere-cube-branch-tree.md)
This method defines a set of procedural voxelization routines used to populate a sparse voxel octree (SVO) data structure with occupancy and color data.

## Operating characteristics
- **Memory Efficiency:** Reduces storage by a factor of 8 for occupancy data compared to 1-bit-per-voxel arrays. Color data also benefits from 8:1 packing.
- **Access Speed:** Bitwise operations are fast, but random access requires division (or shift) and bitwise logic.
- **Scalability:** Memory usage scales linearly with the bounding box volume, regardless of actual voxel count. This is inefficient for extremely sparse scenes where the bounding box is large but occupied volume is small, unless combined with higher-level tree structures (like the parent SVO Tree) to skip empty regions.
- **Limitations:** Requires `sx` to be a multiple of 8. Does not handle hierarchical sparsity itself; that is handled by the parent SVO Tree structure.

## Background
This layout is described in the unofficial repository [sp4cerat/Sparse-Voxel-Octree-Raycasting](https://github.com/sp4cerat/Sparse-Voxel-Octree-Raycasting) [S1, unofficial]. It represents a common implementation detail in early SVO raycasting engines, optimizing the leaf node storage for efficient bitwise ray traversal.

## Sources
[S1] sp4cerat/Sparse-Voxel-Octree-Raycasting, https://github.com/sp4cerat/Sparse-Voxel-Octree-Raycasting, fetched 2026-10-24 (unofficial)
