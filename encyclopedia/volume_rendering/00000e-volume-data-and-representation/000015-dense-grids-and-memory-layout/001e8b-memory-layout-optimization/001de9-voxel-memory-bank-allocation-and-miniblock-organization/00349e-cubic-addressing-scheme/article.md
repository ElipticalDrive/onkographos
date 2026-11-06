# Cubic Addressing Scheme

Also called: Sub-cube addressing

## Overview
The Cubic Addressing Scheme is a memory layout optimization for dense volume rendering that groups voxels into $s^3$ sub-cubes to align with SDRAM row access patterns [S1]. It differs from traditional linear addressing by treating the sub-cube as the fundamental unit of address calculation, thereby reducing the frequency of cache refreshes and SDRAM activation commands during ray casting [S1].

## How it works
**Inputs:**
*   Voxel coordinates $(v_x, v_y, v_z)$.
*   Dataset dimensions $(D_x, D_y, D_z)$.
*   Sub-cube size $s$ (typically 8) [S1].

**Outputs:**
*   Cubic address $A$, which identifies the specific memory location (bank and offset) for the voxel [S1].

**Data Structures:**
The memory is organized into parallel modules (banks). The address $A$ is composed of two parts:
1.  $A_C$: The address of the sub-cube containing the voxel.
2.  $A_V$: The address of the voxel within that sub-cube.

**Parameters:**
*   $s$: The side length of the cubic sub-cube. The typical value is 8 [S1].

**Algorithm:**
The address calculation proceeds in two steps:

1.  **Calculate Sub-cube Index:** Divide the voxel coordinates by $s$ using integer division to determine the coordinates of the sub-cube in the global grid.
    $$
    c_x = \lfloor v_x / s \rfloor, \quad c_y = \lfloor v_y / s \rfloor, \quad c_z = \lfloor v_z / s \rfloor
    $$
2.  **Calculate Intra-sub-cube Offset:** Find the modulus of the voxel coordinates by $s$ to determine the position within the $s^3$ sub-cube.
    $$
    o_x = v_x \pmod s, \quad o_y = v_y \pmod s, \quad o_z = v_z \pmod s
    $$

The final cubic address $A$ is derived from $A_C$ (based on $c_x, c_y, c_z$) and $A_V$ (based on $o_x, o_y, o_z$) [S1]. The specific mapping of $A_C$ and $A_V$ to physical memory banks depends on the hardware architecture (e.g., eight parallel memories), but the logical separation ensures that voxels within the same sub-cube are spatially coherent [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_address(v_x, v_y, v_z, s):
    """
    Calculate the cubic address for a voxel.
    
    Args:
        v_x, v_y, v_z: Integer voxel coordinates.
        s: Sub-cube size (e.g., 8).
        
    Returns:
        A tuple (subcube_coords, intra_subcube_offset) representing the 
        logical address components.
    """
    # Step 1: Calculate sub-cube index
    c_x = v_x // s
    c_y = v_y // s
    c_z = v_z // s
    
    # Step 2: Calculate intra-sub-cube offset
    o_x = v_x % s
    o_y = v_y % s
    o_z = v_z % s
    
    return (c_x, c_y, c_z), (o_x, o_y, o_z)
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The scheme assumes integer coordinates. For floating-point ray casting, coordinates must be floored to the nearest voxel index before applying the scheme [S1].
*   **Cache Size Alignment:** The efficiency relies on the cache size being a multiple of the sub-cube size. For example, with $s=8$, a cache holding $8^3=512$ voxels stores exactly one sub-cube [S1].

## Subtopics

### [Parallel Memory Access with Cubic Addressing](7777-parallel-memory-access-with-cubic-addressing.md)
Extends the scheme to eight parallel memory modules, ensuring trilinear interpolation neighborhoods are served by distinct banks to maximize bandwidth.

## Operating characteristics
**Time and Memory Complexity:**
The address calculation involves constant-time integer division and modulo operations per voxel [S1].

**Measured Performance:**
When comparing cubic addressing to traditional memory addressing (where $A = v_x + n_y v_y + n_y n_z v_z$), the cache hit-to-miss ratio improves significantly for rays aligned with the axes [S1].
*   **Ray casting along x, y, or z axes:** The cache hit-to-miss ratio is 87.5% [S1]. This is because every eighth voxel requires a cache refresh (to load the next sub-cube), resulting in 7 hits and 1 miss per sub-cube traversal [S1].
*   **Traditional Addressing:** Rays parallel to the x-axis achieve 100% cache hit ratio (if cache is filled linearly), but rays parallel to the z-axis suffer 0% hit ratio (cache miss every time) due to the stride [S1].

**Strengths and Limitations:**
*   **Strengths:** Reduces the number of SDRAM activation commands compared to traditional addressing for general ray directions. It aligns well with SDRAM row access patterns [S1].
*   **Limitations:** Performance is still dependent on the specific ray direction relative to the memory layout. It does not eliminate cache misses entirely, unlike some conflict-free skewed organizations [S1].

## Background
The Cubic Addressing Scheme was introduced in the context of low-cost, PCI-based interactive ray casting architectures [S1]. It builds upon previous work in parallel volume rendering where sub-cube addressing was used, but adapts it specifically for SDRAM row constraints [S1]. The method was evaluated using NEC SDRAM characteristics, including parameters like $t_{RCD}$, $t_{RAS}$, and $t_{RP}$ [S1].

## Sources
[S1] A low-cost memory architecture for PCI-based interactive ray casting, https://doi.org/10.1145/311534.311566, fetched 2026-10-27
