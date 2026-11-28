# Block Skewed Memory

Also called: EM-Cube memory organization, Hierarchical Blocks

## Overview
Block Skewed Memory is a memory organization scheme for volume rendering that stores data in cubic blocks of $b \times b \times b$ voxels to optimize DRAM page-mode bandwidth while maintaining conflict-free ray access. It differs from its parent, [Skewed Memory Organization for Conflict-Free Ray Access](2351-skewed-memory-organization-for-conflict-free-ray-access.md), by applying skew at the block granularity rather than the voxel granularity, allowing all voxels of a block to be stored linearly within a single DRAM page [S1, unofficial]. This approach reduces inter-chip communication overhead compared to voxel-level skewing, at the cost of requiring on-chip reordering buffers [S1, unofficial].

## How it works
**Inputs:** Volume data of size $N \times N \times N$ voxels; block dimension $b$; number of memory chips $C$.
**Outputs:** Memory layout mapping voxel coordinates $(x, y, z)$ to memory addresses and chip IDs.

**Data Structures:**
1.  **Block:** A cubic sub-volume of $b^3$ voxels.
2.  **Memory Layout:**
    *   The volume is partitioned into blocks of size $b \times b \times b$.
    *   Blocks are distributed across $C$ memory chips.
    *   Within a chip, blocks are skewed to ensure that any ray accesses at most one voxel per chip per cycle.
    *   Inside a chip, all $b^3$ voxels of a block are stored linearly in contiguous memory addresses (a single DRAM page).
3.  **Reordering Buffer:** An on-chip buffer of size $b^3$ voxels, used by each rendering chip to reorder voxel access within a block according to the current view direction.

**Parameters:**
*   $b$: Block dimension.
    *   **PageBlock:** $b$ is maximized to fit $b^3$ voxels into a single DRAM page. For 512-byte pages: $b=8$ for 8-bit voxels, $b=4$ for 16/32-bit voxels [S1, unofficial].
    *   **MiniBlock:** $b$ is minimized to overlap row access overhead. $b=2$ is typical for 16-bit voxels on 125MHz SDRAM [S1, unofficial].
    *   **Constraint:** $b \leq N/C$ to avoid conflicts for any view direction [S1, unofficial].
*   $C$: Number of memory chips/modules.

**Algorithm Steps:**
1.  **Partition:** Divide the $N \times N \times N$ volume into non-overlapping blocks of size $b \times b \times b$.
2.  **Assign Chips:** Assign each block to a memory chip such that the blocks are skewed. The skew ensures that for any ray direction, the sequence of blocks accessed by the ray maps to distinct chips, preventing bank conflicts.
3.  **Linearize Blocks:** Within each chip, store the $b^3$ voxels of a block in linear, contiguous memory addresses. This allows the DRAM to operate in fast page mode, transferring the entire block (or miniblock) in a single burst.
4.  **Fetch and Reorder:**
    *   When a ray enters a block, the rendering chip fetches the block (or miniblock) from DRAM into the on-chip reordering buffer.
    *   The chip reorders the voxels in the buffer to match the ray's traversal order for the current view direction.
    *   The chip processes the voxels using the Cube-4 algorithm logic, but operating on "partial block-beams" instead of partial beams [S1, unofficial].
5.  **Inter-chip Communication:**
    *   Voxels inside the block are processed on-chip.
    *   Only voxels near the faces of the block require communication with neighboring chips.
    *   The area of a block face is $b^2$. The number of voxels per block is $b^3$.
    *   Inter-chip communication per voxel scales as $1/b$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_block_skewed_layout(N, b, C, voxel_size_bytes):
    """
    Computes the chip ID and offset for a voxel (x, y, z).
    Assumes a specific skew pattern for blocks.
    """
    # Calculate number of blocks per dimension
    # Note: N must be divisible by b for simplicity, or handle boundaries
    blocks_per_dim = N // b
    
    # Determine which block (bx, by, bz) the voxel belongs to
    bx = x // b
    by = y // b
    bz = z // b
    
    # Determine chip ID based on block coordinates and skew
    # A common skew for C chips is: chip_id = (bx + by + bz) % C
    # This ensures that along any diagonal, chips are distinct if C >= 3
    chip_id = (bx + by + bz) % C
    
    # Calculate linear offset within the block
    # Local coordinates within the block
    lx = x % b
    ly = y % b
    lz = z % b
    
    # Linear index within the b^3 block (row-major)
    block_offset = (lz * b * b) + (ly * b) + lx
    
    # Total offset in the chip's memory
    # This assumes blocks are stored sequentially in the chip's memory
    # We need a global block index to determine the chip's base address
    # Global block index (row-major over all blocks)
    global_block_idx = (bz * blocks_per_dim * blocks_per_dim) + \
                       (by * blocks_per_dim) + bx
    
    # Chip memory offset
    # Assuming each chip holds C_th of the blocks, distributed by chip_id
    # This part depends on the specific block-to-chip assignment strategy
    # For a simple round-robin assignment of blocks to chips:
    chip_block_offset = global_block_idx // C
    chip_base_offset = chip_block_offset * (b * b * b)
    
    final_offset = chip_base_offset + block_offset
    
    return chip_id, final_offset

def process_block(chip_id, block_data, view_direction):
    """
    Processes a block of voxels on a rendering chip.
    """
    # 1. Fetch block from DRAM (Page Mode)
    # All b^3 voxels are fetched in one burst
    block_voxels = fetch_from_drampage(chip_id, block_data.address)
    
    # 2. Reorder voxels based on view direction
    # The reordering buffer maps the linear storage to the ray traversal order
    reordered_voxels = reorder_voxels(block_voxels, view_direction)
    
    # 3. Process voxels using Cube-4 logic
    # Handle partial block-beams at the edges of the ray path through the block
    results = process_cube4_style(reordered_voxels, view_direction)
    
    return results
```

**Edge Cases and Pitfalls:**
*   **Block Size Too Large (PageBlock):** Requires a large on-chip buffer ($b^3$ voxels). For 32-bit voxels, $b=4$ requires a 256-voxel buffer. If $b$ exceeds the DRAM page size, the benefit of page mode is lost [S1, unofficial].
*   **Block Size Too Small (MiniBlock):** Increases inter-chip communication overhead. For $b=2$, the face area is 4, but the volume is 8, leading to a high ratio of boundary voxels to internal voxels [S1, unofficial].
*   **Voxel Size Dependency:** PageBlock size $b$ is directly dependent on voxel size. Changing voxel precision (e.g., from 8-bit to 32-bit) may require changing $b$ to fit the DRAM page [S1, unofficial].
*   **Boundary Conditions:** If $N$ is not divisible by $b$, boundary blocks must be handled, potentially requiring padding or special logic [G].

## Subtopics

### PageBlock
Maximizes $b$ to fit a DRAM page. Best for high bandwidth when voxel size is fixed and small. Requires large on-chip buffers.

### MiniBlock
Minimizes $b$ (e.g., $b=2$) to overlap row access latency. Best for reducing on-chip buffer size, but increases inter-chip communication.

### Hierarchical Blocks
Combines PageBlock and MiniBlock by tiling large blocks with small miniblocks. Miniblocks are fetched on demand, reducing buffer requirements while maintaining page-mode efficiency. Blocks are skewed, but miniblocks within a block are not [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Fetching a block is $O(1)$ in terms of DRAM accesses (one page load), but the constant factor depends on $b^3$. Reordering is $O(b^3)$.
*   **Memory Complexity:** On-chip buffer size is $O(b^3)$ per rendering chip.
*   **Inter-chip Communication:** Scales as $1/b$ per voxel. A design with $b=4$ requires up to 4 times less inter-chip communication bandwidth than Cube-4 ($b=1$) [S1, unofficial].
*   **Strengths:** High DRAM bandwidth utilization due to page mode; reduced inter-chip communication compared to voxel-level skewing; view-independent rendering support.
*   **Limitations:** Requires on-chip buffering proportional to $b^3$; block size is sensitive to voxel size (for PageBlock); increased complexity in memory layout and reordering logic.
*   **When to Use:** When DRAM bandwidth is the bottleneck and on-chip buffer resources are available. Suitable for fixed-voxel-size applications.
*   **When Not to Use:** When on-chip buffer space is extremely limited (consider MiniBlock or Hierarchical) or when voxel size varies frequently.

## Background
Block Skewed Memory was introduced in the EM-Cube architecture paper by Harvard's Visual Computing Group in 1997 [S1, unofficial]. It was designed to leverage the high bandwidth of SDRAM in fast page mode for real-time volume rendering. The method addresses the limitations of voxel-level skewed memory (Cube-4) by grouping voxels into blocks, thereby reducing the overhead of inter-chip communication and improving memory access efficiency. The concept of hierarchical blocking was also discussed in related work [12], but EM-Cube distinguished itself by keeping all voxels of a block in the same memory module and using miniblocks for pipelined burst access [S1, unofficial].

## Sources
[S1] EM-Cube: An Architecture for Low-Cost Real-Time Volume Rendering The Harvard community has made this article openly available..., https://vcg.seas.harvard.edu/publications/19970101-em-cube-an-architecture-for-low-cost-real-time-volume-rendering/paper, fetched 2026-10-29 (unofficial)
