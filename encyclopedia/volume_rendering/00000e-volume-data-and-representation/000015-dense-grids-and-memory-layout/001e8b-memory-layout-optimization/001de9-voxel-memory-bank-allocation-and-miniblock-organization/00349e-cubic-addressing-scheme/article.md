# Cubic Addressing Scheme

Also called: Sub-cube addressing

## Overview
The Cubic Addressing Scheme is a memory layout optimization for dense volume rendering that groups voxels into $s^3$ sub-cubes to align with SDRAM row access patterns [S1]. It differs from traditional linear addressing by treating the sub-cube as the fundamental unit of address calculation, thereby reducing the frequency of cache refreshes and SDRAM activation commands during ray casting [S1].

## How it works
**Inputs:**
*   Voxel coordinates $(v_x, v_y, v_z)$ [S1].
*   Dataset dimensions $(D_x, D_y, D_z)$ [S1].
*   Sub-cube size $s$ (typically 8) [S1].

**Outputs:**
*   Cubic address $A$, which identifies the specific memory location for the voxel [S1].

**Data Structures:**
The address $A$ is composed of two parts:
1.  $A_C$: The address of the sub-cube containing the voxel.
2.  $A_V$: The address of the voxel within that sub-cube.

**Parameters:**
*   $s$: The side length of the cubic sub-cube. The typical value is 8 [S1].

**Algorithm:**
The address calculation proceeds in two steps:

1.  **Calculate Sub-cube Index:** Divide the voxel coordinates by $s$ to find the $s^3$ sub-cube [S1].
    $$
    c_x = \lfloor v_x / s \rfloor, \quad c_y = \lfloor v_y / s \rfloor, \quad c_z = \lfloor v_z / s \rfloor
    $$
2.  **Calculate Intra-sub-cube Offset:** Find the modulus by $s$ of the coordinates to determine the position within the sub-cube [S1].
    $$
    o_x = v_x \pmod s, \quad o_y = v_y \pmod s, \quad o_z = v_z \pmod s
    $$

The address $A$ combines $A_C$ and $A_V$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cubic_address(v_x, v_y, v_z, s):
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
*   **Boundary Conditions:** The scheme operates on integer voxel coordinates [S1].
*   **Cache Size Alignment:** With $s=8$, a cache of 512 voxels stores exactly one sub-cube [S1].

## Subtopics

### [Parallel Memory Access with Cubic Addressing](7777-parallel-memory-access-with-cubic-addressing.md)
Extends the scheme to eight parallel memory modules, ensuring trilinear interpolation neighborhoods are served by distinct banks to maximize bandwidth.

## Operating characteristics
**Time and Memory Complexity:**
The address calculation involves constant-time integer division and modulo operations per voxel [S1].

**Measured Performance:**
When comparing cubic addressing to traditional memory addressing, the cache hit-to-miss ratio improves for rays aligned with the axes [S1].
*   **Ray casting along x, y, or z axes:** The cache hit-to-miss ratio is 87.5% [S1]. Every eighth voxel requires a cache refresh, resulting in 7 hits and 1 miss per sub-cube traversal [S1].
*   **Traditional Addressing:** Rays parallel to the x-axis achieve a 100% cache hit ratio when the cache is filled linearly, but rays parallel to the z-axis suffer a 0% hit ratio due to the memory stride [S1].

**Strengths and Limitations:**
*   **Strengths:** Reduces the number of SDRAM activation commands compared to traditional addressing for general ray directions [S1].
*   **Limitations:** Performance varies with the specific ray direction relative to the memory layout [S1].

## Background
The Cubic Addressing Scheme was introduced for low-cost, PCI-based interactive ray casting architectures [S1]. It builds upon prior parallel volume rendering work using sub-cube addressing, but adapts it specifically for SDRAM row constraints [S1]. The method was evaluated using NEC SDRAM characteristics, including parameters like $t_{RCD}$, $t_{RAS}$, and $t_{RP}$ [S1].

## Sources
[S1] A low-cost memory architecture for PCI-based interactive ray casting, https://doi.org/10.1145/311534.311566, fetched 2026-10-27
