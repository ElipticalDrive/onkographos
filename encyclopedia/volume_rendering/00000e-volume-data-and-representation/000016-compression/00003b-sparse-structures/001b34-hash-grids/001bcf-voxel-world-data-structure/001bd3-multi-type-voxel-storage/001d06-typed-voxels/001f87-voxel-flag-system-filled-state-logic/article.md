# Voxel Flag System & Filled State Logic

Also called: Bitmask Voxel States, VoxelMeta State Machine

## Overview
The Voxel Flag System manages the topological and geometric state of individual voxels within a sparse grid by maintaining a compact bitmask of five distinct properties. It solves the problem of efficiently determining whether a voxel is part of the solid volume, its surface, or empty space without requiring separate metadata arrays for each property. It differs from [Typed Voxels](1399-typed-voxels.md) by focusing on binary geometric classification (solid/surface/inside) rather than material type enumeration, allowing for rapid updates during mesh voxelization.

## How it works
**Inputs and Outputs:**
- Input: A mesh (vertices and triangles) and a target grid resolution.
- Output: A sparse grid where each active voxel contains a `VoxelMeta` structure.

**Data Structures:**
The core state is stored in a 5-bit bitmask defined by `VoxelFlagsEnum`. The layout is:
- `FILLED`: Bit 0 (`0x01 << 0`)
- `INSIDE`: Bit 1 (`0x01 << 1`)
- `SURFACE`: Bit 2 (`0x01 << 2`)
- `SOLID`: Bit 3 (`0x01 << 3`)
- `CLIPPED`: Bit 4 (`0x01 << 4`)

Each voxel in the grid is represented by a `VoxelMeta` struct containing:
- `index_`: The linear or 3D index of the voxel in the grid.
- `flags_`: A `uint8_t` or `uint32_t` integer storing the `VoxelFlagsEnum` bitmask.

**Algorithm Steps:**
1. **Initialization:** Call `ClearFlags()` to reset all flags to 0. Set the `INSIDE` bit to 1 for all voxels initially, assuming a default "inside" state for the bounding volume [S1, unofficial].
2. **Voxelization:** For each triangle in the mesh, determine which voxels it intersects.
   - If a voxel is intersected by a triangle, set the `SOLID` flag.
   - If a voxel is on the boundary of the mesh (intersected but not fully contained), set the `SURFACE` flag.
   - If a voxel is outside the mesh bounds or clipped by the grid boundary, set the `CLIPPED` flag [S1, unofficial].
3. **State Propagation:** Whenever any flag (`INSIDE`, `SURFACE`, `SOLID`, `CLIPPED`) is modified, call `UpdateFilled()` to recalculate the `FILLED` state.
4. **Filled State Calculation:** The `FILLED` bit is computed based on the following logic:
   - A voxel is `FILLED` if it is **not** `CLIPPED` AND (`SOLID` OR (`SURFACE` AND `INSIDE`)).
   - Otherwise, `FILLED` is unset.
5. **Querying:** To determine the physical state, check the `FILLED` bit. If set, the voxel is part of the solid volume.

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
- Grid resolution: Defined by `grid_size_` (x, y, z dimensions).
- Voxel size: Derived from mesh bounds and grid size.
- Epsilon: `kEpsBox` is added to mesh bounds to prevent floating-point errors at boundaries [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in intersection tests can lead to incorrect `SOLID` or `SURFACE` flags. Using an epsilon box around the mesh bounds helps mitigate this [S1, unofficial].
- **Degenerate Input:** Triangles with zero area or invalid indices should be filtered out during mesh loading to avoid undefined behavior in voxelization [S1, unofficial].
- **Clipping:** Voxels outside the defined grid bounds must be marked as `CLIPPED` to prevent out-of-bounds access or incorrect state propagation.

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
        self.SetInside()  # Initialize INSIDE bit

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
        # FILLED = !CLIPPED && (SOLID || (SURFACE && INSIDE))
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
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, assuming constant-time flag updates. Voxelization step depends on mesh complexity and grid resolution.
- **Memory Complexity:** $O(1)$ per voxel (5 bits + index). Sparse storage reduces memory usage for large empty regions.
- **Strengths:** Compact memory footprint, fast state updates, simple logic for determining filled status.
- **Limitations:** Limited to 5 states; not suitable for complex material properties or multi-layered voxels.
- **When to Use:** Ideal for real-time voxelization, collision detection, and simple physics simulations where geometric state is primary.
- **When Not to Use:** When detailed material properties, color, or multi-type data per voxel are required.

## Background
The Voxel Flag System originates from the `topskychen/voxelizer` project, an open-source C++ library for converting 3D meshes into voxel grids. It was developed to provide an efficient way to manage voxel states during the voxelization process, ensuring that geometric properties are accurately captured and updated. The system is used in applications requiring fast voxel-based collision detection and rendering, such as game engines and simulation software.

## Sources
[S1] topskychen/voxelizer, https://github.com/topskychen/voxelizer, fetched 2026-10-27 (unofficial)
