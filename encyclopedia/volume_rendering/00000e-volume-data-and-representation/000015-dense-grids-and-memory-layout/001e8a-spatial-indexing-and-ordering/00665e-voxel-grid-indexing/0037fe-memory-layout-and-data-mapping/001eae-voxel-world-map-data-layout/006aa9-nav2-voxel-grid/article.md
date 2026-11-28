# Nav2 Voxel Grid


## Overview
The Nav2 Voxel Grid is an efficient 3D spatial indexing structure designed for robotics navigation, specifically serving as the backend for the Voxel Layer in `nav2_costmap_2d` [S1, S2]. It solves the problem of representing 3D occupancy data with minimal memory overhead by mapping a 3D world onto a 2D array of pointers, where each pointer manages a vertical column of voxels [S1, S2]. It distinguishes itself from general-purpose 3D grids by relying on bitwise integer operations to achieve raytracing and cell marking performance comparable to standard 2D structures, making it significantly faster than most 3D structures [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Map dimensions (width, height), voxel resolution, and vertical limits [S1, S2].
*   **Output:** A data structure supporting 3D raycasting and cell marking (marked, free, unknown) [S1, S2].

**Data Structures:**
The method implements a dense voxel grid where the horizontal dimensions ($X, Y$) are managed via a 2D array, and the vertical dimension ($Z$) is managed via bitmasks within integer types [S1].
*   **Pointer Array:** Each element `grid[x][y]` points to a memory block representing the vertical column at $(x, y)$ [S1, S2].
*   **Voxel State Encoding:** The state of each voxel in the column is encoded using bit locations within integer types [S1, S2].
*   **Cell States:** The grid supports three distinct representations for the state of a cell: marked, free, or unknown [S1, S2].

**Numbered Steps:**
1.  **Initialization:** Allocate a 2D array of pointers with size equal to the map width $\times$ height [S1, S2]. Initialize each pointer to `nullptr` or an empty block [G].
2.  **Column Access:** To access a voxel at $(x, y, z)$, retrieve the column pointer and extract the bit corresponding to $z$ [G].
3.  **Cell Marking:** To mark a voxel as occupied, free, or unknown, use bitwise OR (`|`) to set the corresponding bit or bitwise AND (`&`) to clear it [S1, S2].
4.  **Raycasting:** Perform 3D raycasting by iterating through voxel column indices along the ray path, checking the bitmask state for intersected levels [G].

**Formulas:**
*   Bitmask for level $z$: $M_z = 1 \ll z$, where $0 \le z < 16$ [G].
*   Mark occupied: $C_{\text{new}} = C_{\text{old}} \mid M_z$ [G].
*   Mark free/unknown: $C_{\text{new}} = C_{\text{old}} \ \& \ \sim M_z$ [G].
*   Check state: $\text{occupied} = (C \ \& \ M_z) \neq 0$ [G].

**Parameters:**
*   **Levels per Column:** The grid supports up to 16 different levels per voxel column [S1, S2]. This limit is a direct consequence of the underlying implementation relying on bitwise AND and OR integer operations [S1, S2].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Level Limit:** The implementation is strictly limited to 16 levels per voxel column due to the reliance on bitwise integer operations [S1, S2].
*   **Precision:** The use of bitwise operations requires careful handling of integer widths to avoid overflow or incorrect state interpretation [G].
*   **Boundaries:** Accessing coordinates outside the $W \times H$ map bounds or $Z$ levels outside $[0, 15]$ must be guarded [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Nav2VoxelGrid:
    def __init__(self, width, height, levels_per_column=16):
        self.width = width
        self.height = height
        self.levels = levels_per_column
        # 2D array of integers (bitmasks) representing columns
        self.grid = [[None for _ in range(height)] for _ in range(width)]

    def set_voxel(self, x, y, z, state):
        if not (0 <= x < self.width and 0 <= y < self.height):
            raise IndexError("Out of bounds")
        if not (0 <= z < self.levels):
            raise ValueError(f"Z level {z} exceeds max levels {self.levels}")

        if self.grid[x][y] is None:
            self.grid[x][y] = 0

        mask = 1 << z
        if state == "marked":
            self.grid[x][y] |= mask
        elif state == "free" or state == "unknown":
            self.grid[x][y] &= ~mask
            if self.grid[x][y] == 0:
                self.grid[x][y] = None
        else:
            raise ValueError("Invalid state")

    def get_voxel(self, x, y, z):
        if not (0 <= x < self.width and 0 <= y < self.height):
            return "unknown"
        if not (0 <= z < self.levels):
            return "unknown"

        col = self.grid[x][y]
        if col is None:
            return "unknown"
        if col & (1 << z):
            return "marked"
        return "free"

    def raycast(self, origin, direction, max_dist):
        # Standard 3D DDA iteration through grid cells
        x, y, z = float(origin[0]), float(origin[1]), float(origin[2])
        dx, dy, dz = float(direction[0]), float(direction[1]), float(direction[2])
        steps = int(max_dist / max(abs(dx), abs(dy), abs(dz))) if max(abs(dx), abs(dy), abs(dz)) > 0 else 0
        for _ in range(steps):
            ix, iy, iz = int(x), int(y), int(z)
            if 0 <= ix < self.width and 0 <= iy < self.height and 0 <= iz < self.levels:
                col = self.grid[ix][iy]
                if col is not None and (col & (1 << iz)):
                    return (ix, iy, iz, "marked")
            x += dx
            y += dy
            z += dz
        return None
```

## Operating characteristics
*   **Time Complexity:** Raytracing and cell marking performance are comparable to standard 2D structures [S1, S2]. This is significantly faster than most general 3D structures [S1, S2].
*   **Memory Complexity:** Memory usage is proportional to the map area ($W \times H$) [S1, S2]. Each column uses a small integer to store up to 16 levels [S1, S2].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with the number of voxels traversed during raycasting, but the 2D pointer array overhead is minimal compared to full 3D array allocations [G].
*   **Strengths:** High speed for raycasting and marking; simple implementation using bitwise operations [S1, S2].
*   **Limitations:** Hard limit of 16 vertical levels per column; limited to 3 cell states (marked, free, unknown) [S1, S2].
*   **When to use:** When high-performance 3D occupancy grid operations (raycasting, marking) are required in robotics navigation, particularly within the ROS 2 Nav2 ecosystem [S1, S2].
*   **When not to use:** When more than 16 vertical levels are needed, or when more than 3 distinct cell states are required [S1, S2].

## Background
The Nav2 Voxel Grid package originated as a direct port to ROS 2 for use in the voxel layer of `nav2_costmap_2d` [S1, S2]. It was branched out as a separate package to allow reuse in other applications where a dense voxel grid representation may be useful [S1, S2]. It is used by the Voxel Layer inside `nav2_costmap_2d` [S1, S2]. The package also contains implementations of 3D raycasting [S1, S2].

## Sources
[S1] nav2_voxel_grid — nav2_voxel_grid: Kilted 1.4.2 documentation, https://docs.ros.org/en/kilted/p/nav2_voxel_grid/, fetched 2026-10-24
[S2] nav2_voxel_grid — nav2_voxel_grid: Humble 1.1.20 documentation, https://docs.ros.org/en/humble/p/nav2_voxel_grid/, fetched 2026-10-24
[S3] nav2_voxel_grid — nav2_voxel_grid: Lyrical 1.5.3 documentation, https://docs.ros.org/en/lyrical/p/nav2_voxel_grid/, fetched 2026-10-24
