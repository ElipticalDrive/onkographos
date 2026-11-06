# Two-Level Sparse Voxel Grid Layout

Also called: BrickMap, BrickPool

## Overview
Two-Level Sparse Voxel Grid Layout stores a sparse voxel world using an outer grid of cells that point to fixed-size inner bricks of packed voxels [S1, unofficial]. It differs from its parent [Brick-based Sparse Voxel Lookup with Open-Addressing Hash](2587-brick-based-sparse-voxel-lookup-with-open-addressing-hash.md) by replacing the open-addressing hash table with a fixed-size outer grid, trading hash collision resolution for predictable memory layout and simpler boundary checks [S1, unofficial].

## How it works
**Inputs:** World dimensions in voxels (`WORLDSIZE`), brick dimensions (`BRICKSIZE`), and a set of active voxel coordinates with associated data (e.g., material ID, density).
**Outputs:** An outer grid array and a dynamic pool of brick structures.

**Data Structures:**
1.  **Outer Grid:** A 3D array of size `GRIDSIZE³`. Each cell stores either a null pointer or a pointer to a brick.
    *   `GRIDSIZE = WORLDSIZE / BRICKSIZE` [S1, unofficial].
    *   Default `WORLDSIZE = 512`, default `BRICKSIZE = 8`, resulting in `GRIDSIZE = 64` [S1, unofficial].
2.  **Brick:** A contiguous block of memory containing `BRICKSIZE³` packed voxels.
    *   Default voxel representation: `uint32_t` [S1, unofficial].
    *   Size: $8 \times 8 \times 8 = 512$ voxels.

**Algorithm:**
1.  Initialize the outer grid of size `GRIDSIZE³` with null pointers.
2.  Initialize an empty dynamic brick pool.
3.  For each active voxel at world coordinates $(x, y, z)$ with data $D$:
    a. Calculate brick coordinates:
       $$ bx = \lfloor x / BRICKSIZE \rfloor $$
       $$ by = \lfloor y / BRICKSIZE \rfloor $$
       $$ bz = \lfloor z / BRICKSIZE \rfloor $$
    b. Clamp $(bx, by, bz)$ to the range $[0, GRIDSIZE-1]$ to handle boundary conditions [G].
    c. Access the outer grid cell at index $(bx, by, bz)$.
    d. If the cell is null:
       i. Allocate a new brick from the dynamic pool.
       ii. Initialize the brick's voxel array to empty/default values.
       iii. Store the pointer to the new brick in the outer grid cell.
       iv. Safety clamp: Ensure the pool does not exceed `GRIDSIZE³` bricks [S1, unofficial].
    e. Calculate local voxel coordinates within the brick:
       $$ lx = x \pmod{BRICKSIZE} $$
       $$ ly = y \pmod{BRICKSIZE} $$
       $$ lz = z \pmod{BRICKSIZE} $$
    f. Store $D$ in the brick's voxel array at index $(lx, ly, lz)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
WORLDSIZE = 512
BRICKSIZE = 8
GRIDSIZE = WORLDSIZE // BRICKSIZE  # 64

# Data Structures
class Brick:
    voxels: list[uint32_t]  # Size BRICKSIZE^3

class Grid:
    cells: list[Brick | None]  # Size GRIDSIZE^3

def init_world() -> Grid:
    grid = Grid()
    grid.cells = [None] * (GRIDSIZE ** 3)
    return grid

def add_voxel(grid: Grid, x: int, y: int, z: int, data: uint32_t):
    # Calculate brick coordinates
    bx = x // BRICKSIZE
    by = y // BRICKSIZE
    bz = z // BRICKSIZE
    
    # Clamp to grid bounds
    bx = max(0, min(bx, GRIDSIZE - 1))
    by = max(0, min(by, GRIDSIZE - 1))
    bz = max(0, min(bz, GRIDSIZE - 1))
    
    # Calculate linear index in outer grid
    idx = (bz * GRIDSIZE + by) * GRIDSIZE + bx
    
    # Allocate brick if needed
    if grid.cells[idx] is None:
        grid.cells[idx] = Brick()
        # Initialize voxels to default (e.g., 0 or empty)
        grid.cells[idx].voxels = [0] * (BRICKSIZE ** 3)
        
    # Calculate local coordinates
    lx = x % BRICKSIZE
    ly = y % BRICKSIZE
    lz = z % BRICKSIZE
    
    # Store data in brick
    local_idx = (lz * BRICKSIZE + ly) * BRICKSIZE + lx
    grid.cells[idx].voxels[local_idx] = data

def get_voxel(grid: Grid, x: int, y: int, z: int) -> uint32_t:
    bx = x // BRICKSIZE
    by = y // BRICKSIZE
    bz = z // BRICKSIZE
    
    # Clamp to grid bounds
    bx = max(0, min(bx, GRIDSIZE - 1))
    by = max(0, min(by, GRIDSIZE - 1))
    bz = max(0, min(bz, GRIDSIZE - 1))
    
    idx = (bz * GRIDSIZE + by) * GRIDSIZE + bx
    
    brick = grid.cells[idx]
    if brick is None:
        return 0  # Empty voxel
        
    lx = x % BRICKSIZE
    ly = y % BRICKSIZE
    lz = z % BRICKSIZE
    
    local_idx = (lz * BRICKSIZE + ly) * BRICKSIZE + lx
    return brick.voxels[local_idx]
```

**Parameters:**
*   `WORLDSIZE`: World dimension in voxels. Default: 512 [S1, unofficial].
*   `BRICKSIZE`: Brick dimension in voxels. Default: 8 [S1, unofficial].
*   `GRIDSIZE`: Outer grid dimension. Calculated as `WORLDSIZE / BRICKSIZE`. Default: 64 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Voxels at the world boundary must be clamped to the outer grid indices to prevent out-of-bounds access [G].
*   **Pool Capacity:** The dynamic brick pool is safety-clamped to `GRIDSIZE³` to prevent infinite growth [S1, unofficial].
*   **Memory Fragmentation:** Using a dynamic pool may lead to fragmentation if bricks are frequently allocated and deallocated. A contiguous pool or arena allocator is recommended [G].

## Subtopics

### Brick with Sparse Top-Level
A variant using 8×8×8 voxel bricks compressed with an occupancy mask and material palette, organized under a sparse top-level index (hash map or shallow 64-tree) instead of a dense outer grid [S2]. It trades edit triviality for excellent streaming/disk unit properties and lower memory [S2]. Recommended architecture matching Gustafsson's next engine (2024) and van Wingerden's brickmaps [S2]. Per-brick compact cost is ~210 B (64 B mask + ~128 B palette + ~18 B header), yielding ~3.3 B per surface voxel [S2].

## Operating characteristics
*   **Time Complexity:** Lookup and insertion are $O(1)$ average case, assuming constant-time arithmetic for coordinate calculation.
*   **Memory Complexity:** Outer grid consumes $GRIDSIZE^3 \times \text{pointer\_size}$ bytes. Brick pool consumes $N_{bricks} \times BRICKSIZE^3 \times \text{voxel\_size}$ bytes. Total memory is proportional to the number of active bricks, not the total world volume.
*   **Performance:** Not given in the sources.
*   **Strengths:** Simple implementation, predictable memory layout for the outer grid, efficient for sparse worlds.
*   **Limitations:** Outer grid memory is fixed and may be wasteful if the world is very large but sparse. Hash-based variants may offer better memory efficiency for extremely sparse worlds [S2].
*   **When to use:** When the world size is moderate and the sparsity is not extreme, or when implementation simplicity is prioritized.
*   **When not to use:** When the world is extremely large and sparse, a hash-based top level may be more memory efficient [S2].

## Background
Originated in the context of voxel-based game engines and ray tracers, specifically referenced in the `bad-apple-voxel-raytracer` project [S1, unofficial]. The concept of "BrickMap" has been explored in various indie game development communities, including references to Gustafsson's engine and van Wingerden's implementations [S2].

## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S2] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
