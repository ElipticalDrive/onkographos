# Interleaved Data Distribution
Also called: Page-based Round-robin Distribution
## Overview
Interleaved Data Distribution is a memory layout strategy for shared-memory MIMD architectures that distributes pages of a voxel array in round-robin fashion among local processor memories [S1]. It solves the problem of memory hot spotting, where multiple processors simultaneously access the same memory bank during ray tracing, by ensuring that adjacent voxels are stored in different local memories [S1].
## How it works
**Inputs and Outputs:**
- Input: A 3D voxel dataset.
- Input: The number of local memory banks $P$ (processors).
- Output: A mapping function that assigns each voxel to a specific local memory bank.

**Data Structures:**
Voxels are stored in a linearized array structure [S1]. The array is divided into contiguous blocks called pages, which are distributed in round-robin fashion among the local memories [S1]. The page size is not given in the sources.

**Steps:**
1. Linearize the 3D voxel grid into a 1D array.
2. Divide the linear array into pages.
3. Assign each page to a local memory bank using round-robin distribution [S1].
4. Store the voxels belonging to each page in its assigned local memory bank [S1].

**Formulas:**
- Linear Index: Not given in the sources. [G] Standard row-major layout: $i = x + y \cdot N_x + z \cdot N_x \cdot N_y$.
- Page Index: Not given in the sources. [G] $p = \lfloor i / B \rfloor$.
- Memory Bank: Not given in the sources. [G] $m = p \mod P$.

**Parameters:**
- $P$: Number of processors/local memories.
- $B$: Page size. Not given in the sources.

**Edge Cases and Pitfalls:**
- The sources do not specify how to handle dataset dimensions that are not multiples of the page size or number of banks. [G] Remaining voxels are distributed to the next available banks in round-robin order.
- The sources do not specify optimal page size trade-offs. [G] If $B$ is too small, management overhead may outweigh benefits; if $B$ is too large, distribution granularity decreases, potentially causing hot spotting within a page.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assign_voxel_to_memory(x, y, z, Nx, Ny, B, P):
    """
    Assigns a voxel at (x, y, z) to a local memory bank.
    
    Args:
        x, y, z: Voxel coordinates.
        Nx, Ny: Dimensions of the voxel grid.
        B: Page size in voxels.
        P: Number of local memory banks.
        
    Returns:
        The index of the local memory bank.
    """
    # Step 1: Linearize the 3D index
    i = x + y * Nx + z * Nx * Ny
    
    # Step 2: Calculate page index
    p = i // B
    
    # Step 3: Round-robin assignment to memory bank
    m = p % P
    
    return m
```
## Subtopics

### [Grouped vs Interleaved Macro Distribution](13385-grouped-vs-interleaved-macro-distribution.md)
Classifies how a node's allocated macros are spatially arranged within the global voxel dataset. It addresses the spatial locality of data chunks to optimize cache coherence and manage inter-node communication during ray casting. It sets itself apart by defining the geometric arrangement of the redistribution unit, whereas sibling methods define the size or routing of that unit. It is chosen when inter-node communication patterns must align with macro boundaries.

## Operating characteristics
**Time and Memory Complexity:**
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.

**Measured Performance:**
- The method was implemented on the DASH (Directory Architecture for SHared memory) Multiprocessor [S1].
- It avoids hot spotting, which improves scalability compared to static assignment [S1].

**Scaling:**
- The sources do not specify scaling conditions. [G] Scales with dataset size as long as the page size is chosen appropriately.

**Strengths and Limitations:**
- Strengths: Avoids hot spotting [S1].
- Limitations: Not given in the sources.

**When to Use:**
- Not given in the sources.

**When Not to Use:**
- Not given in the sources.
## Background
**Origin:**
- Proposed in "Volume rendering on scalable shared-memory MIMD architectures" [S1].

**History and Development:**
- Implemented on the DASH Multiprocessor [S1].

**Where it is Used:**
- Shared-memory MIMD architectures [S1].
## Sources
[S1] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
