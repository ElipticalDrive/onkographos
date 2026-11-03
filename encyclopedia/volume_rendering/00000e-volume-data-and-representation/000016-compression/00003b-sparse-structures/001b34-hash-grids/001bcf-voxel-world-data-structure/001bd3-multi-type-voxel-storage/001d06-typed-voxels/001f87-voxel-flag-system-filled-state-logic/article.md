# Voxel Flag System & Filled State Logic

Also called: Bitmask Voxel States, VoxelMeta State Machine

## Overview
The Voxel Flag System manages the topological and geometric state of individual voxels within a sparse grid by maintaining a compact bitmask of five distinct properties [S1, unofficial]. It solves the problem of efficiently determining whether a voxel is part of the solid volume, its surface, or empty space without requiring separate metadata arrays for each property [S1, unofficial]. It differs from [Typed Voxels](1399-typed-voxels.md) by focusing on binary geometric classification (solid/surface/inside) rather than material type enumeration, allowing for rapid updates during mesh voxelization [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A mesh (vertices and triangles) and a target grid resolution [S1, unofficial].
- Output: A sparse grid where each active voxel contains a `VoxelMeta` structure [S1, unofficial].

**Data Structures:**
The core state is stored in a 5-bit bitmask defined by `VoxelFlagsEnum` [S1, unofficial]. The layout is:
- `FILLED`: Bit 0 (`0x01 << 0`)
- `INSIDE`: Bit 1 (`0x01 << 1`)
- `SURFACE`: Bit 2 (`0x01 << 2`)
- `SOLID`: Bit 3 (`0x01 << 3`)
- `CLIPPED`: Bit 4 (`0x01 << 4`)

Each voxel in the grid is represented by a `VoxelMeta` struct containing:
- `index_`: The linear or 3D index of the voxel in the grid [S1, unofficial].
- `flags_`: An integer storing the `VoxelFlagsEnum` bitmask [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Call `ClearFlags()` to reset all flags to 0 and set the `INSIDE` bit [S1, unofficial].
2. **Voxelization:** Determine which voxels each mesh triangle intersects and set the `SOLID` or `SURFACE` flags accordingly. Not given in the sources.
3. **State Propagation:** Whenever any flag (`INSIDE`, `SURFACE`, `SOLID`, `CLIPPED`) is modified, call `UpdateFilled()` to recalculate the `FILLED` state [S1, unofficial].
4. **Filled State Calculation:** The `FILLED` bit is computed based on the following logic:
   - A voxel is `FILLED` if it is **not** `CLIPPED` AND (`SOLID` OR (`SURFACE` AND `INSIDE`)) [S1, unofficial].
   - Otherwise, `FILLED` is unset.
5. **Querying:** To determine the physical state, check the `FILLED` bit. If set, the voxel is part of the solid volume [S1, unofficial].

**Formulas:**
The filled state $F$ is calculated as:
$$ F = \neg C \land (S \lor (U \land I)) $$
Where:
- $F$: FILLED bit
- $C$: CLIPPED bit
- $S$: SOLID bit
- $U$: SURFACE bit
- $I$: INSIDE bit

**Parameters:**
- Grid resolution: Defined by `grid_size_` (x, y, z dimensions) [S1, unofficial].
- Voxel size: Derived from mesh bounds and grid size [S1, unofficial].
- Epsilon: `kEpsBox` is added to mesh bounds during initialization [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Clipping:** Voxels outside the defined grid bounds must be marked as `CLIPPED` to prevent out-of-bounds access or incorrect state propagation [S1, unofficial].
- **Precision:** Floating-point errors in intersection tests can affect flag accuracy [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelFlagsEnum:
    FILLED = 0x01 << 0
    INSIDE = 0x01 << 1
    SURFACE = 0x01 << 2
    SOLID = 0x01 << 3
    CLIPPED = 0x01 << 4

class VoxelMeta:
    def __init__(self, index):
        self.index_ = index
        self.flags_ = 0
        self.SetInside()

    def ClearFlags(self):
        self.flags_ = 0
        self.SetInside()

    def SetInside(self):
        self.flags_ |= VoxelFlagsEnum.INSIDE
        self.UpdateFilled()

    def UnsetInside(self):
        self.flags_ &= ~VoxelFlagsEnum.INSIDE
        self.UpdateFilled()

    def SetSurface(self):
        self.flags_ |= VoxelFlagsEnum.SURFACE
        self.UpdateFilled()

    def UnsetSurface(self):
        self.flags_ &= ~VoxelFlagsEnum.SURFACE
        self.UpdateFilled()

    def SetSolid(self):
        self.flags_ |= VoxelFlagsEnum.SOLID
        self.UpdateFilled()

    def UnsetSolid(self):
        self.flags_ &= ~VoxelFlagsEnum.SOLID
        self.UpdateFilled()

    def SetClipped(self):
        self.flags_ |= VoxelFlagsEnum.CLIPPED
        self.UpdateFilled()

    def UnsetClipped(self):
        self.flags_ &= ~VoxelFlagsEnum.CLIPPED
        self.UpdateFilled()

    def UpdateFilled(self):
        is_clipped = bool(self.flags_ & VoxelFlagsEnum.CLIPPED)
        is_solid = bool(self.flags_ & VoxelFlagsEnum.SOLID)
        is_surface = bool(self.flags_ & VoxelFlagsEnum.SURFACE)
        is_inside = bool(self.flags_ & VoxelFlagsEnum.INSIDE)

        if not is_clipped and (is_solid or (is_surface and is_inside)):
            self.flags_ |= VoxelFlagsEnum.FILLED
        else:
            self.flags_ &= ~VoxelFlagsEnum.FILLED

    def IsFilled(self):
        return bool(self.flags_ & VoxelFlagsEnum.FILLED)
```

## Subtopics

### [Voxel Material Classification](2882-voxel-material-classification.md)
Determines physical interaction type (solid, liquid, air) by unpacking the packed integer representation. It differs from this method by focusing on immediate binary classification for simulation logic rather than geometric state management.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** $O(1)$ per voxel (5 bits + index) [S1, unofficial].
- **Strengths:** Compact memory footprint and fast state updates [S1, unofficial].
- **Limitations:** Limited to 5 states [S1, unofficial].
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
The Voxel Flag System originates from the `topskychen/voxelizer` project, an open-source C++ library for converting 3D meshes into voxel grids [S1, unofficial]. It was developed to provide an efficient way to manage voxel states during the voxelization process, ensuring that geometric properties are accurately captured and updated [S1, unofficial].

## Sources
[S1] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
