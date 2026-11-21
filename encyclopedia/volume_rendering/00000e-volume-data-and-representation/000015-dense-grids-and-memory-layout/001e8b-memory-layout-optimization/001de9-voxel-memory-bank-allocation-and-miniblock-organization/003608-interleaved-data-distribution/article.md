# Interleaved Data Distribution
Also called: Page-based Round-robin Distribution
## Overview
Interleaved Data Distribution is a memory layout strategy for shared-memory MIMD architectures that distributes pages of a voxel array in round-robin fashion among local processor memories [S1]. It solves the problem of memory hot spotting, where multiple processors simultaneously access the same memory bank during ray tracing, by ensuring that adjacent voxels are stored in different local memories [S1]. It sets itself apart from its siblings by using a simple linearized array with round-robin page assignment, whereas [Skewed Memory Organization for Conflict-Free Ray Access](2351-skewed-memory-organization-for-conflict-free-ray-access.md) uses coordinate-based skewing and [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md) uses spatial locality optimization [S1].
## How it works
**Inputs and Outputs:**
- Input: A 3D voxel dataset of size $N_x \times N_y \times N_z$.
- Input: The number of local memory banks $P$ (processors).
- Output: A mapping function that assigns each voxel to a specific local memory bank.

**Data Structures:**
The voxel data is stored in a single linearized array of size $N = N_x \times N_y \times N_z$. The array is divided into contiguous blocks called "pages". The size of a page is not explicitly defined in the sources, but it is a fixed block size $B$ [S1].

**Steps:**
1. Linearize the 3D voxel grid into a 1D array $V$ of size $N$. The index $i$ of a voxel $(x, y, z)$ is calculated as $i = x + y \cdot N_x + z \cdot N_x \cdot N_y$ [S1].
2. Divide the linear array $V$ into pages of size $B$. The page index $p$ for a voxel at linear index $i$ is $p = \lfloor i / B \rfloor$ [S1].
3. Assign each page $p$ to a local memory bank $m$ using round-robin distribution: $m = p \mod P$ [S1].
4. Store the voxels belonging to page $p$ in the local memory bank $m$ [S1].

**Formulas:**
- Linear Index: $i(x, y, z) = x + y \cdot N_x + z \cdot N_x \cdot N_y$
- Page Index: $p(i) = \lfloor i / B \rfloor$
- Memory Bank: $m(p) = p \mod P$

**Parameters:**
- $P$: Number of processors/local memories.
- $B$: Page size. Not given in the sources.

**Edge Cases and Pitfalls:**
- If $B$ is too small, the overhead of managing many small pages may outweigh the benefits.
- If $B$ is too large, the granularity of the round-robin distribution decreases, potentially leading to hot spotting within a page if all voxels in a page are accessed by the same processor.
- The sources do not specify how to handle the case where the total number of voxels is not a multiple of $P \cdot B$. [G] Assume remaining voxels are distributed to the next available banks in round-robin order.

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
Classifies how a node's allocated macros are spatially arranged within the global voxel dataset. It sets itself apart by defining the geometric arrangement of the redistribution unit (macro), whereas [Macro Distribution Granularity](16651-macro-distribution-granularity.md) defines the size of that unit.

## Operating characteristics
**Time and Memory Complexity:**
- Time Complexity: $O(1)$ for memory bank lookup.
- Memory Complexity: $O(N)$ for storing the voxel data, distributed across $P$ banks.

**Measured Performance:**
- The method was implemented on the DASH (Directory Architecture for SHared memory) Multiprocessor [S1].
- It avoids hot spotting, which improves scalability compared to static assignment [S1].

**Scaling:**
- Scales with dataset size as long as the page size $B$ is chosen appropriately [S1].

**Strengths and Limitations:**
- Strengths: Simple to implement, avoids hot spotting, scalable [S1].
- Limitations: Does not optimize for spatial locality, which may lead to cache misses if adjacent voxels are accessed frequently [S1].

**When to Use:**
- Use when memory hot spotting is a primary concern and spatial locality is less critical.

**When Not to Use:**
- Do not use when spatial locality is critical for performance, such as in ray casting with high coherence.

## Background
**Origin:**
- Proposed in "Volume rendering on scalable shared-memory MIMD architectures" [S1].

**History and Development:**
- Implemented on the DASH Multiprocessor [S1].

**Where it is Used:**
- Shared-memory MIMD architectures [S1].

## Sources
[S1] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
