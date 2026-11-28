# Voxel Memory Bank Allocation and Miniblock Organization

Also called: Miniblock Reordering for SDRAM, Voxel Memory Layout, Voxel Memory Organization (Miniblocks and Skewing)

## Overview
Voxel Memory Bank Allocation and Miniblock Organization is a hardware-oriented memory layout strategy designed to maximize the throughput of SDRAM-based volume rendering systems by ensuring conflict-free, burst-mode memory access [S1, unofficial]. It solves the problem of memory bank conflicts and page misses that typically stall rendering pipelines when rays traverse 3D voxel grids [S1, unofficial]. It sets itself apart from siblings like [Skewed Memory Organization](2351-skewed-memory-organization-for-conflict-free-ray-access.md) by operating at the granularity of 2x2x2 voxel sub-cubes (miniblocks) rather than individual voxels, and from [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md) by explicitly managing the mapping of these miniblocks to physical SDRAM banks to prevent internal bank collisions [S1, unofficial].

## How it works
The method organizes voxel data into fixed-size blocks, distributes them across multiple memory modules, and applies a skewing transformation to align with the physical structure of SDRAM chips [S1, unofficial].

### Inputs and Outputs
*   **Input:** A 3D volume dataset of arbitrary dimensions $D_u \times D_v \times D_w$ stored in host memory [S1, unofficial].
*   **Output:** A layout in Voxel Memory where voxels are arranged in miniblocks, distributed across memory modules, and skewed to memory banks, ready for burst reads by rendering pipelines [S1, unofficial].

### Data Structures and Layout
1.  **Miniblock:** A 2x2x2 array of voxels. It is the fundamental unit of memory access [S1, unofficial]. Miniblocks are stored linearly in a single memory module at consecutive memory addresses [S1, unofficial].
2.  **Voxel Memory Dimensions:** The allocated memory dimensions must be multiples of 32 voxels in each direction ($2 \times 4 \times 4 = 32$) to ensure proper alignment for SDRAM burst performance [S1, unofficial]. Arbitrary host volumes are padded to these dimensions and automatically cropped during rendering [S1, unofficial].
3.  **Memory Modules:** The system uses four memory modules (interfaces) [S1, unofficial].
4.  **SDRAM Banks:** Each SDRAM chip contains four internal memory banks (e.g., 64-megabit SDRAMs) [S1, unofficial].

### Coordinate Mapping and Formulas
Voxels with coordinates $(u, v, w)$ are mapped to miniblock coordinates $(u_m, v_m, w_m)$:
$$
u_m = \lfloor u / 2 \rfloor, \quad v_m = \lfloor v / 2 \rfloor, \quad w_m = \lfloor w / 2 \rfloor
$$
where $\lfloor \cdot \rfloor$ denotes integer division [S1, unofficial].

The memory module index $M$ and memory bank index $B$ are calculated as follows:
$$
M \equiv (u_m + v_m + w_m) \bmod 4
$$
$$
B \equiv ((u_m \div 4) + (v_m \div 4) + (w_m \div 4)) \bmod 4
$$
where $\div$ is integer division and $\bmod$ is the modulus operator [S1, unofficial].

### Algorithm Steps
1.  **Padding:** Allocate Voxel Memory with dimensions rounded up to the nearest multiple of 32 in each axis [S1, unofficial]. Copy the source volume data into this buffer, padding with zeros or boundary values as needed [S1, unofficial].
2.  **Miniblock Formation:** Group voxels into 2x2x2 miniblocks [S1, unofficial]. Store each miniblock sequentially in memory to exploit SDRAM burst capabilities [S1, unofficial].
3.  **Module Distribution:** Distribute miniblocks across the four memory modules such that any group of four adjacent miniblocks (along any axis) resides in separate modules [S1, unofficial]. This is achieved by the module index formula above [S1, unofficial].
4.  **Bank Skewing:** Within each memory module, skew the position of miniblocks so that adjacent miniblocks map to different SDRAM banks [S1, unofficial]. This prevents bank conflicts during back-to-back accesses [S1, unofficial]. This is achieved by the bank index formula above [S1, unofficial].
5.  **Reordering (De-skewing):** During rendering, the view direction changes [S1, unofficial]. The Voxel Memory controllers read miniblocks in their stored order [S1, unofficial]. A global de-skewing network re-shuffles the four miniblocks output by the memory interfaces so that the left-most voxel along the x-axis flows down the leftmost pipeline, and adjacent x-voxels flow down neighboring pipelines [S1, unofficial].
6.  **Slice Buffering:** The de-skewed miniblocks are written into voxel slice buffers in the rendering pipelines [S1, unofficial]. The interface reads two slices of voxels at a time to populate the buffers [S1, unofficial].
7.  **Gradient Computation:** The slice buffers store three slices worth of data to allow z-gradient computation using central differences [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def allocate_voxel_memory(volume_data, dims_u, dims_v, dims_w):
    # Step 1: Padding to multiples of 32
    alloc_u = ((dims_u + 31) // 32) * 32
    alloc_v = ((dims_v + 31) // 32) * 32
    alloc_w = ((dims_w + 31) // 32) * 32

    # Initialize 4 memory modules
    modules = [None] * 4
    for i in range(4):
        modules[i] = [0] * (alloc_u * alloc_v * alloc_w)

    # Step 2, 3, 4: Map voxels to miniblocks, modules, and banks
    for u in range(dims_u):
        for v in range(dims_v):
            for w in range(dims_w):
                voxel = volume_data[u, v, w]

                # Calculate miniblock coordinates
                um = u // 2
                vm = v // 2
                wm = w // 2

                # Calculate Module Index
                module_idx = (um + vm + wm) % 4

                # Calculate Bank Index within the module
                bank_idx = ((um // 4) + (vm // 4) + (wm // 4)) % 4

                # Determine offset within the 2x2x2 miniblock (8 voxels)
                offset_in_mb = (u % 2) * 4 + (v % 2) * 2 + (w % 2)

                # Calculate linear address of the miniblock within the module
                # Sources do not specify the exact linear mapping algorithm for miniblocks within a module;
                # a standard row-major mapping is assumed for implementation.
                mb_linear_addr = (um * alloc_v * alloc_w + vm * alloc_w + wm) * 8

                # Write voxel to memory
                modules[module_idx][mb_linear_addr + offset_in_mb] = voxel

    return modules, alloc_u, alloc_v, alloc_w

def render_pipeline_step(modules, alloc_u, alloc_v, alloc_w):
    # Step 5: De-skewing
    # The hardware de-skewing network rearranges the 4 miniblocks from the 4 modules
    # so that the left-most voxel along the x-axis flows down the leftmost pipeline,
    # and adjacent x-voxels flow down neighboring pipelines [S1, unofficial].

    # Step 6: Slice Buffering
    # The Voxel Memory interface reads two slices of voxels at a time because of the memory organization in miniblocks [S1, unofficial].
    # Sources do not specify the exact slice extraction algorithm; standard 2D slice extraction is used.
    slice_buffer = []
    for module in modules:
        slice_buffer.append(module[0:alloc_u * alloc_v])

    # Step 7: Gradient Computation
    # The slice buffers store three slices worth of data so that the z gradients can be computed using central differences [S1, unofficial].
    # Sources do not specify the exact buffer refill algorithm; standard circular buffer logic is used.
    while len(slice_buffer) < 3:
        slice_buffer.append(module[0:alloc_u * alloc_v])

    # Compute interpolated samples and gradients from the buffered slices
    # Standard tri-linear interpolation and central difference gradient calculation
    return 0.0, (0.0, 0.0, 0.0)
```

### Parameters and Edge Cases
*   **Volume Dimensions:** Must be treated as multiples of 32 voxels. If the original volume is smaller, it is padded [S1, unofficial].
*   **SDRAM Type:** The design assumes SDRAMs with 4 internal banks (e.g., 64-megabit chips) [S1, unofficial].
*   **Supervolumes:** For volumes exceeding 256 voxels on any edge, software must partition the volume into blocks (supervolumes), render them independently, and blend the results [S1, unofficial]. Adjacent blocks duplicate 3 voxels in each dimension to maintain correct interpolation and gradients at boundaries [S1, unofficial].
*   **Pitfall:** Incorrect alignment of volume dimensions relative to the 32-voxel boundary will degrade SDRAM burst performance [S1, unofficial].

## Subtopics

### [Skewed Memory Organization for Conflict-Free Ray Access](2351-skewed-memory-organization-for-conflict-free-ray-access.md)
Skewed memory organization is a hardware-oriented memory layout for Cubic Frame Buffers (CFBs) that maps 3D voxel data into $n$ memory banks such that any ray of $n$ voxels accesses exactly one voxel per bank, enabling conflict-free simultaneous retrieval in a single memory cycle. It sets itself apart from standard interleaved or tiled layouts by using linear modular arithmetic to guarantee parallel access for specific viewing directions, often combined with barrel shifting to support efficient geometric transformations.

### [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md)
The Cubic Addressing Scheme is a memory layout optimization for dense volume rendering that groups voxels into $s^3$ sub-cubes to align with SDRAM row access patterns. It differs from traditional linear addressing by treating the sub-cube as the fundamental unit of address calculation, thereby reducing the frequency of cache refreshes and SDRAM activation commands during ray casting.

### [Eight Logical Memories Architecture](7287-eight-logical-memories-architecture.md)
The Eight Logical Memories Architecture is a memory layout optimization for dense volume rendering that maps voxel data to physical SDRAM banks to minimize access latency. It solves the problem of high switching times by grouping neighboring subcubes into a logical unit that spans two physical SDRAM devices, assuming four banks per SDRAM.

### [Interleaved Data Distribution](7660-interleaved-data-distribution.md)
Interleaved Data Distribution is a memory layout strategy for shared-memory MIMD architectures that distributes pages of a voxel array in round-robin fashion among local processor memories. It solves the problem of memory hot spotting, where multiple processors simultaneously access the same memory bank during ray tracing, by ensuring that adjacent voxels are stored in different local memories. It sets itself apart from its siblings by using a simple linearized array with round-robin page assignment, whereas [Skewed Memory Organization for Conflict-Free Ray Access](2351-skewed-memory-organization-for-conflict-free-ray-access.md) uses coordinate-based skewing and [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md) uses spatial locality optimization.

### [Dual-Port Memory & Synchronization Architecture](11730-dual-port-memory-synchronization-architecture.md)
The Dual-Port Memory & Synchronization Architecture is a hardware interface method for 3-D volumetric displays that transforms 2D image points into 3D voxels by synchronizing memory readout with the physical position of a moving display surface (helix or piston). It differs from sibling methods like [Skewed Memory Organization](2351-skewed-memory-organization-for-conflict-free-ray-access.md) or [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md) by relying on external mechanical synchronization signals rather than static memory layout tricks to resolve spatial mapping, allowing direct memory mapping for high-speed host writes.

### Sixty Four Memories Architecture
An ideal solution using 64 independently accessible SDRAMs to achieve conflict-free memory access without pipeline stalls [S2]. It ensures no access penalty while crossing a page boundary by placing 2x2 neighborhoods across the page boundary in four of the other 12 SDRAMs not currently accessed, and precharging/activating SDRAMs 4-15 while accessing SDRAMs 1-4 so new voxels are available within the next clock cycle [S2]. It requires the calculation and distribution of 64 independent addresses, which is expensive to build and exceeds the board space budget for a PCI-Card [S2]. This architecture reaches the optimum of 10 ns access time and a 100% cache hit ratio [S2].

### Double-buffered Voxel Storage and Routing
A graphics memory module using DDR SDRAM divided into three colors (Red, Green, Blue) and two buffers (active/inactive) to buffer the entire volume of data for continuous display [S3]. It schedules and controls scanning of an 'active' buffer, random reads and writes to an 'inactive' buffer, and refreshes via a graphics memory manager [S3]. It routes data between the CPU, graphics memory, and DMDs using three voxel routers (one per color) [S3]. It implements automatic read-modify-write capability in voxel routers to handle logic operations between new data and data already being displayed [S3]. It initiates each read-modify-write operation with only one write instruction from the Raster Engine [S3]. It performs AND, OR, XOR, and masking operations during read-modify-write [S3]. It buffers all instructions in a FIFO in the FPGA to be performed when scheduling allows [S3]. Total capacity is 6 Gbits DDR SDRAM, divided into 6 SDRAM buffers of 128 MB each (3 colors x 2 buffers) [S3].

### Balanced Bank Allocation for Feature Cache
A storage strategy for feature vectors that maps vertex IDs to Feature SRAM (FSRAM) banks to equalize storage distribution and maintain uniform depth across banks, resolving imbalanced distribution issues [S4]. Feature vectors for an entire fine voxel are stored collectively instead of separately for each micro voxel, reducing the number of feature vectors needing storage per fine voxel from 512 to 125 [S4]. Features are stored in eight FSRAMs based on vertex IDs (0-7), with each voxel vertex containing a feature vector categorized into eight types according to vertex ID [S4]. Direct spatial indices are used as addresses [S4]. Direct mapping of vertex IDs to FSRAM IDs results in non-uniform depth of FSRAM due to imbalanced distribution of feature vectors across vertex types within a fine voxel, increasing placement and routing difficulty [S4].

## Operating characteristics
*   **Time Complexity:** Memory access is $O(1)$ per miniblock, with no pipeline stalls for conflict-free accesses [S1, unofficial][S2].
*   **Memory Complexity:** Requires padding volumes to multiples of 32 voxels, increasing memory usage slightly for non-aligned dimensions [S1, unofficial].
*   **Performance:** On the VolumePro hardware, four 133 MHz memory interfaces can read Voxel Memory at a burst rate of 532 million voxels per second, provided voxels are organized appropriately [S1, unofficial]. The 64-memory ideal architecture achieves 10 ns access time and 100% cache hit ratio, but is impractical for PCI cards [S2]. The 4-DIMM/8-logical-memory alternative offers virtually no loss in performance (<1 fps difference) compared to the 64-memory ideal, with high practicality [S2].
*   **Strengths:** Maximizes SDRAM burst efficiency; eliminates bank conflicts for adjacent miniblocks; supports real-time rendering of large volumes (up to 128 million 8-bit voxels) [S1, unofficial].
*   **Limitations:** Requires specific hardware support (de-skewing network, miniblock reorder logic) [S1, unofficial]; limited to volumes with max 256 voxels per edge without software supervolume partitioning [S1, unofficial].

## Background
The method was developed for the VolumePro real-time ray-casting system, described in the 1999 paper "The VolumePro Real-Time Ray-Casting System" [S1, unofficial]. It addresses the memory bandwidth and latency bottlenecks of early GPU/ASIC-based volume rendering hardware [S1, unofficial]. The concept of using 64 independent memories as an ideal baseline was discussed in "A low-cost memory architecture for PCI-based interactive ray casting" [S2].

## Sources
[S1] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S2] A low-cost memory architecture for PCI-based interactive ray casting, https://doi.org/10.1145/311534.311566, fetched 2026-10-27
[S3] Volumetric three-dimensional display system with rasterization hardware, https://doi.org/10.1117/12.430821, fetched 2026-11-01
[S4] An Energy-Efficient Edge Coprocessor for Neural Rendering With Explicit Data Reuse Strategies, https://doi.org/10.1109/tvlsi.2025.3641653, fetched 2026-10-26
