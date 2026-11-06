# Voxel Memory Bank Allocation and Miniblock Organization

Also called: Miniblock Reordering for SDRAM, Voxel Memory Layout, Voxel Memory Organization (Miniblocks and Skewing)

## Overview
Voxel Memory Bank Allocation and Miniblock Organization is a hardware-oriented memory layout technique designed to maximize SDRAM burst read throughput and eliminate memory bank conflicts during dense volume rendering [S1, unofficial]. It solves the problem of inefficient memory access patterns by grouping voxels into small, fixed-size units (miniblocks) and applying modular arithmetic to distribute these units across memory modules and banks [S1, unofficial]. Unlike standard linear addressing, this method ensures that adjacent miniblocks in any spatial direction map to distinct memory resources, allowing for conflict-free, back-to-back access regardless of the ray traversal direction [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset with dimensions $(U, V, W)$.
*   **Output:** A linear memory layout in Voxel Memory where voxels are organized into miniblocks, distributed across memory modules, and skewed within banks.

**Data Structures and Layout:**
1.  **Miniblock:** A $2 \times 2 \times 2$ array of voxels. These 8 voxels are stored linearly in a single memory module at consecutive memory addresses to exploit SDRAM burst capabilities [S1, unofficial].
2.  **Voxel Memory Dimensions:** The host volume is padded or allocated such that all dimensions are multiples of 32 voxels ($2 \times 4 \times 4 = 32$) [S1, unofficial]. This alignment ensures miniblocks are properly aligned for SDRAM performance [S1, unofficial].
3.  **Cropping:** Arbitrary volumes in host memory are stored in Voxel Memory with these padded dimensions and are automatically cropped to their original size during rendering [S1, unofficial].

**Coordinate Mapping:**
Voxel coordinates $(u, v, w)$ are mapped to miniblock coordinates $(u_m, v_m, w_m)$ using integer division by 2:
$$ u_m = \lfloor u / 2 \rfloor, \quad v_m = \lfloor v / 2 \rfloor, \quad w_m = \lfloor w / 2 \rfloor $$
The memory module index $M$ and bank index $B$ are calculated as follows [S1, unofficial]:
$$ M = (u_m + v_m + w_m) \bmod 4 $$
$$ B = ((u_m \div 4) + (v_m \div 4) + (w_m \div 4)) \bmod 4 $$
where $\div$ denotes integer division and $\bmod$ denotes the modulus operator [S1, unofficial].

**Algorithm Steps:**
1.  **Allocation:** Allocate Voxel Memory with dimensions that are multiples of 32 voxels in each axis [S1, unofficial].
2.  **Miniblock Formation:** Group voxels into $2 \times 2 \times 2$ miniblocks. Store each miniblock sequentially in a single memory module [S1, unofficial].
3.  **Module Distribution:** Distribute miniblocks across 4 memory modules such that any group of 4 adjacent miniblocks in any direction (x, y, or z) maps to separate modules. This is achieved by the module index formula $M = (u_m + v_m + w_m) \bmod 4$ [S1, unofficial].
4.  **Bank Skewing:** Within each memory module, skew the placement of miniblocks so that adjacent miniblocks never fall into the same memory bank. This is achieved by the bank index formula $B = ((u_m \div 4) + (v_m \div 4) + (w_m \div 4)) \bmod 4$ [S1, unofficial].
5.  **Readout and Reordering:** When reading, the Voxel Memory interface reads two slices of voxels at a time (comprising multiple miniblocks) [S1, unofficial]. Miniblocks are read out in the order they are stored (sequential) to maintain burst mode [S1, unofficial].
6.  **De-skewing and Shuffling:** A global de-skewing network rearranges the four miniblocks output by the memory controllers based on the current view direction [S1, unofficial]. This ensures that the left-most voxel along the x-axis flows down the leftmost pipeline and adjacent voxels in x flow down neighboring pipelines, matching the canonical positions required for tri-linear interpolation [S1, unofficial].
7.  **Buffering:** The de-skewed miniblocks are written into voxel slice buffers in four rendering pipelines [S1, unofficial]. The system stores three slices worth of data in these buffers to compute z-gradients using central differences [S1, unofficial].

**Parameters:**
*   **Miniblock Size:** $2 \times 2 \times 2$ voxels [S1, unofficial].
*   **Memory Modules:** 4 modules [S1, unofficial].
*   **Memory Banks per SDRAM:** 4 banks (based on 64-megabit SDRAMs) [S1, unofficial].
*   **Volume Dimension Alignment:** Multiples of 32 voxels [S1, unofficial].
*   **Slice Buffer Depth:** 3 slices (to support central difference gradient calculation) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Handling:** The method assumes volumes are padded to multiples of 32. Arbitrary volumes must be cropped during rendering to avoid accessing invalid memory [S1, unofficial].
*   **Interpolation Requirements:** The layout supports tri-linear interpolation by ensuring that the 8 voxels of a miniblock are accessible in a specific order after de-skewing [S1, unofficial].
*   **Gradient Calculation:** Central differences for z-gradients require data from three consecutive slices, necessitating the storage of 3 slices in the pipeline buffers [S1, unofficial].

## Subtopics

### [Skewed Memory Organization for Conflict-Free Ray Access](2351-skewed-memory-organization-for-conflict-free-ray-access.md)
Skewed memory organization is a hardware-oriented memory layout for Cubic Frame Buffers (CFBs) that maps 3D voxel data into $n$ memory banks such that any ray of $n$ voxels accesses exactly one voxel per bank, enabling conflict-free simultaneous retrieval in a single memory cycle. It sets itself apart from standard interleaved or tiled layouts by using linear modular arithmetic to guarantee parallel access for specific viewing directions, often combined with barrel shifting to support efficient geometric transformations. Link: [Skewed Memory Organization for Conflict-Free Ray Access](2351-skewed-memory-organization-for-conflict-free-ray-access.md).

### [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md)
The Cubic Addressing Scheme is a memory layout optimization for dense volume rendering that groups voxels into $s^3$ sub-cubes to align with SDRAM row access patterns. It differs from traditional linear addressing by treating the sub-cube as the fundamental unit of address calculation, thereby reducing the frequency of cache refreshes and SDRAM activation commands during ray casting. Link: [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md).

### [Eight Logical Memories Architecture](7287-eight-logical-memories-architecture.md)
The Eight Logical Memories Architecture is a memory layout optimization for dense volume rendering that maps voxel data to physical SDRAM banks to minimize access latency. It solves the problem of bank conflicts and high switching times by grouping eight neighboring subcubes into a logical unit that spans two physical SDRAM devices, assuming four banks per SDRAM. It sets itself apart from the parent [Voxel Memory Bank Allocation and Miniblock Organization](1638-voxel-memory-bank-allocation-and-miniblock-organization.md) by introducing a hierarchical grouping of banks into logical memories, and from its sibling [Skewed Memory Organization for Conflict-Free Ray Access](2351-skewed-memory-organization-for-conflict-free-ray-access.md) by relying on coordinated precharging rather than spatial skewing to handle ray traversal. Link: [Eight Logical Memories Architecture](7287-eight-logical-memories-architecture.md).

### Sixty Four Memories Architecture
An ideal solution using 64 independently accessible SDRAMs to achieve conflict-free memory access without pipeline stalls [S2]. It ensures no access penalty while crossing a page boundary by placing $2 \times 2$ neighborhoods across the page boundary in four of the other 12 SDRAMs not currently accessed [S2]. It precharges and activates SDRAMs 4-15 while accessing SDRAMs 1-4, so new voxels are available within the next clock cycle when crossing the page boundary [S2]. This architecture reaches the optimum of 10 ns access time and a 100% cache hit ratio [S2]. However, it requires the calculation and distribution of 64 independent addresses, which is expensive to build and exceeds the board space budget for a PCI-Card [S2].

### Interleaved Data Distribution
A data distribution strategy for shared-memory MIMD architectures that distributes pages of the voxel array in round-robin fashion among local memories to avoid hot spotting [S3]. Voxels are stored in a linearized array structure, and pages of this array are distributed in round-robin fashion among the local memories of the processors [S3].

## Operating characteristics
*   **Time Complexity:** Memory access is $O(1)$ per miniblock, with constant-time arithmetic for module and bank indexing [S1, unofficial].
*   **Memory Complexity:** Requires padding volume dimensions to multiples of 32 voxels, increasing memory usage slightly for non-aligned volumes [S1, unofficial].
*   **Performance:** Enables burst reads of 8 or more voxels (miniblock size) at rates up to 532 million voxels per second with four 133 MHz memory interfaces [S1, unofficial].
*   **Strengths:** Eliminates memory bank conflicts for adjacent miniblocks in any direction; maximizes SDRAM burst efficiency; supports real-time rendering on hardware like VolumePro [S1, unofficial].
*   **Limitations:** Requires specific hardware support for de-skewing and reordering; limited to volumes with dimensions that can be padded to multiples of 32; maximum edge size of 256 voxels per supervolume block due to on-chip buffer limits [S1, unofficial].
*   **When to Use:** High-performance, hardware-accelerated volume rendering where SDRAM bandwidth is a bottleneck and ray traversal directions are arbitrary [S1, unofficial].
*   **When Not to Use:** Software rendering on general-purpose CPUs where cache locality is managed differently; scenes requiring dynamic memory layouts that cannot be pre-padded [S1, unofficial].

## Background
The Voxel Memory Bank Allocation and Miniblock Organization technique was developed for the VolumePro real-time ray-casting system [S1, unofficial]. It was introduced in a 1999 technical report by the authors of the VolumePro system to address the challenges of reading large volume datasets (up to 128 million 8-bit voxels) from SDRAM at high speeds [S1, unofficial]. The method builds on earlier work in memory skewing for graphics hardware [S1, unofficial]. It is used in the VolumePro hardware architecture, specifically in the vg500 ASIC, which contains the necessary hardware to arrange voxels into miniblocks and perform skewing and de-skewing during data transfers [S1, unofficial].

## Sources
[S1] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S2] A low-cost memory architecture for PCI-based interactive ray casting, https://doi.org/10.1145/311534.311566, fetched 2026-10-27
[S3] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
