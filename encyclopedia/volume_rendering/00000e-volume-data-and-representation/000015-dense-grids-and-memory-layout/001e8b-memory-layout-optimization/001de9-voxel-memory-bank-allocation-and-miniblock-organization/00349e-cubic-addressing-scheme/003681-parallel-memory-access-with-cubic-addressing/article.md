# Parallel Memory Access with Cubic Addressing
Also called: Memory Access Buffering (FIFO)

## Overview
Parallel Memory Access with Cubic Addressing is a hardware-aware memory layout and access strategy for interactive ray casting that distributes voxel data across eight parallel memory modules (banks) to maximize bandwidth and minimize pipeline stalls [S1]. It builds on [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md) by ensuring that each trilinear interpolation neighborhood is served by distinct memory banks, allowing simultaneous access to all eight corner voxels [S1]. Unlike unbuffered approaches where frame time depends on the cumulative latency of all memory accesses, this method decouples memory retrieval from the rendering pipeline using FIFO buffers, making performance dependent only on the slowest individual SDRAM bank [S1].

## How it works
**Inputs:**
- Ray sample coordinates $(V_z, V_y, V_x)$ in the volume grid.
- Sub-cube size $s$ (typically a power of two, e.g., 8 or 16, though exact $s$ is "Not given in the sources" for this specific method, it is implied by the parent article).
- Eight parallel memory modules $M_0$ through $M_7$.

**Outputs:**
- Eight voxel values retrieved from the eight memory modules for trilinear interpolation.
- Updated memory addresses for the next sample point.

**Data Structures:**
- **Memory Layout:** The volume is partitioned into $s^3$ sub-cubes. Each sub-cube is distributed across the eight memory banks. Within each bank, voxels are stored such that each bank holds every second voxel along specific axes to facilitate the addressing scheme [S1].
- **FIFO Buffers:** For each of the eight memory modules, there are two FIFO buffers: one for addresses and one for voxel values [S1].

**Address Calculation:**
The core of the method is calculating the address for each of the eight memory modules $M_i$ based on the sample point $(V_z, V_y, V_x)$. Since each memory module stores only every second voxel, the base coordinates are divided by two (implemented as a bitwise shift) [S1].

Let $A$ be the address for a specific memory module. The address is calculated as:
$$ A = s^3 AC + A_v $$
where $AC$ is the sub-cube address and $A_v$ is the voxel address within the sub-cube [S1].

The voxel address components $A_v$ depend on the memory module index $i$ and the sample coordinates. The coordinates used for address calculation are derived from $V_x, V_y, V_z$. If the sample point coordinates are divisible by two, the coordinates used are the original coordinates divided by two [S1].

For a sample point $S$, the modified coordinates for each memory module are calculated as follows [S1]:

$$
\begin{aligned}
M_0 &: (M_x, M_y, M_z) \\
M_1 &: (M_x, M_y, M_z) \\
M_2 &: (M_x, M_y, M_z) \\
M_3 &: (M_x, M_y, M_z) \\
M_4 &: (M_x, M_y, M_z) \\
M_5 &: (M_x, M_y, M_z) \\
M_6 &: (M_x, M_y, M_z) \\
M_7 &: (M_x, M_y, M_z)
\end{aligned}
$$

*Note: The source text provides a fragmented formula for $M_0$ and $M_1$ and states "where..." but does not provide the complete explicit formulas for all eight modules in the provided excerpts. It states that for $M_0$, the x-coordinate is divided by two and the modulus by two of the x-coordinate is added, and for $M_1$, only the division by two is needed [S1]. The general pattern involves an address modification unit (incrementer) when the sample point is between two neighborhoods [S1].*

**Pipeline Operation:**
1.  **Address Generation:** Calculate the eight memory addresses for the current sample point. This involves dividing the sample address by two and applying modulus operations to determine the specific voxel within the sub-cube for each bank [S1].
2.  **FIFO Enqueue:** Push the calculated addresses into the address FIFO buffers for each of the eight memory modules [S1].
3.  **Memory Access:** The memory controllers read from the SDRAMs. The voxel values are pushed into the voxel value FIFO buffers [S1].
4.  **Interpolation:** The ray casting pipeline consumes voxel values from the voxel FIFO buffers to perform trilinear interpolation [S1].

**Edge Cases and Pitfalls:**
-   **Cache Misses and Pipeline Stalls:** The pipeline stalls only when the voxel FIFO of a particular memory is empty at the same time that this memory requires a cache refill [S1].
-   **Sub-cube Boundary Crossing:** When a ray crosses the y-z plane of the next $s^3$ sub-cube, memories $M_0, M_2, M_4, M_6$ change coordinates and refresh their cache, while $M_1, M_3, M_5, M_7$ remain in the current sub-cube [S1]. If cache refreshes happen in separate pipeline cycles, the pipeline stalls twice [S1].
-   **Consecutive Cache Misses:** Depending on the ray direction, a particular memory can have up to 2 cache misses consecutively [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
NUM_BANKS = 8
# s is the sub-cube size, assumed to be a power of two
# s = ... # Not given in the sources

def calculate_memory_addresses(vz, vy, vx):
    """
    Calculates the memory addresses for the 8 parallel banks
    for a given sample point (vz, vy, vx).
    """
    addresses = []
    
    # Base division by 2 (shift operation)
    # Note: The source implies integer division or bitwise shift
    base_x = vx >> 1
    base_y = vy >> 1
    base_z = vz >> 1
    
    # Modulo operations to determine offset within the sub-cube
    mod_x = vx % 2
    mod_y = vy % 2
    mod_z = vz % 2
    
    # The source provides specific logic for M0 and M1 and mentions
    # an address modification unit (incrementer) for boundaries.
    # The exact formulas for all 8 banks are not fully detailed in the source.
    # We implement the described logic for M0 and M1 and generalize
    # based on the "divisible by two" rule and bank distribution.
    
    for i in range(NUM_BANKS):
        # Determine the specific coordinate adjustments for bank i
        # This is a simplified representation based on the text:
        # "for memory M0... x coordinate must be divided by two and 
        # the modulus by two of the x coordinate added to it"
        # "for memory M1... x coordinate only needs to be divided by two"
        
        # Placeholder for bank-specific logic not fully explicit in source
        # In practice, this involves mapping the 8 voxels of the trilinear
        # neighborhood to the 8 banks based on their parity in x, y, z.
        
        # Example for M0 (based on source snippet):
        if i == 0:
            ax = base_x + mod_x
            ay = base_y
            az = base_z
        # Example for M1 (based on source snippet):
        elif i == 1:
            ax = base_x
            ay = base_y
            az = base_z
        else:
            # General case: The source states "If (Vz, Vy, Vx) is divisible 
            # by two, the coordinates used... are the original coordinates 
            # divided by two."
            # And "address modification unit... before each memory bank"
            # for boundary cases.
            # Without the full formula, we assume a standard parity mapping
            # for cubic addressing where each bank corresponds to a specific
            # (x%2, y%2, z%2) combination.
            # This is a gap filled by [G] based on standard cubic addressing
            # principles, as the source text is fragmented.
            parity_x = vx % 2
            parity_y = vy % 2
            parity_z = vz % 2
            
            # Map bank index to parity (standard for 8 banks)
            # Bank i corresponds to bits (i&1, (i>>1)&1, (i>>2)&1)
            bx = parity_x
            by = parity_y
            bz = parity_z
            
            ax = base_x + bx
            ay = base_y + by
            az = base_z + bz
            
        # Calculate final address
        # A = s^3 * AC + Av
        # AC is the sub-cube address (vz//s, vy//s, vx//s)
        # Av is the voxel address within the sub-cube
        # This part is generic to Cubic Addressing, not specific to Parallel Access
        # but required for implementation.
        subcube_z = vz // s
        subcube_y = vy // s
        subcube_x = vx // s
        subcube_addr = subcube_z * s * s + subcube_y * s + subcube_x
        
        # Voxel address within subcube depends on the specific bank's layout
        # The source does not provide the exact Av formula for all banks.
        # We assume a standard linear mapping within the subcube for the bank.
        # This is a gap filled by [G].
        voxel_addr = (az * s * s) + (ay * s) + ax
        
        addresses.append(subcube_addr * (s*s*s) + voxel_addr)
        
    return addresses

def ray_cast_pipeline(rays, volume_data):
    """
    Simulates the ray casting pipeline with FIFO buffering.
    """
    # Initialize FIFOs for each of the 8 banks
    # Each FIFO has a capacity, e.g., 4 entries
    address_fifos = [FIFO(capacity=4) for _ in range(NUM_BANKS)]
    value_fifos = [FIFO(capacity=4) for _ in range(NUM_BANKS)]
    
    for ray in rays:
        for sample in ray.samples:
            vz, vy, vx = sample.coordinates
            
            # 1. Calculate addresses
            mem_addrs = calculate_memory_addresses(vz, vy, vx)
            
            # 2. Enqueue addresses
            for i in range(NUM_BANKS):
                address_fifos[i].enqueue(mem_addrs[i])
                
            # 3. Memory Access (Simulated)
            # In hardware, this happens in parallel.
            # We check if FIFOs are empty and need refill.
            for i in range(NUM_BANKS):
                if value_fifos[i].is_empty():
                    # Check if address FIFO has an address
                    if not address_fifos[i].is_empty():
                        addr = address_fifos[i].dequeue()
                        # Read from volume data (simulating SDRAM access)
                        voxel = volume_data.read(addr)
                        value_fifos[i].enqueue(voxel)
                    else:
                        # Pipeline stall: FIFO empty and no address to fetch
                        # This is the only stall condition described
                        pass 
                        
            # 4. Interpolation
            # Dequeue values for trilinear interpolation
            voxels = []
            for i in range(NUM_BANKS):
                if not value_fifos[i].is_empty():
                    voxels.append(value_fifos[i].dequeue())
                else:
                    # Should not happen if pipeline is balanced
                    voxels.append(0.0)
            
            # Perform trilinear interpolation
            color = trilinear_interpolate(voxels, sample.t)
            
    return color
```

## Operating characteristics
-   **Time Complexity:** The frame time is dependent on the performance of the slowest SDRAM bank, rather than the sum of all access times [S1]. This is a significant improvement over unbuffered memory access [S1].
-   **Memory Complexity:** Requires eight parallel memory modules (or logical banks) and FIFO buffers for addresses and voxel values for each module [S1].
-   **Performance:** Provides optimal performance with poor feasibility in terms of board space if 64 independent addresses are calculated (as in a 64-SDRAM solution) [S1]. The 8-bank solution is more practical and fits on a PC1 board, though it may suffer from unbalanced access times if not carefully designed [S1].
-   **Strengths:** Minimizes pipeline stalling due to cache misses by buffering [S1]. Allows for high bandwidth utilization by accessing 8 voxels in parallel [S1].
-   **Limitations:** Pipeline stalls can still occur if the FIFOs are empty and a cache refill is needed [S1]. Stalls can be compounded if cache refreshes happen in separate pipeline cycles (e.g., when crossing sub-cube boundaries) [S1].

## Background
This method was introduced in the context of low-cost memory architectures for PCI-based interactive ray casting [S1]. It addresses the limitations of traditional memory addressing and unbuffered parallel access by combining cubic addressing with parallel memory access and FIFO buffering [S1]. The work highlights the trade-off between optimal performance (requiring many memory modules) and practical feasibility (using fewer modules with banking) [S1].

## Sources
[S1] A low-cost memory architecture for PCI-based interactive ray casting, https://doi.org/10.1145/311534.311566, fetched 2026-10-27
