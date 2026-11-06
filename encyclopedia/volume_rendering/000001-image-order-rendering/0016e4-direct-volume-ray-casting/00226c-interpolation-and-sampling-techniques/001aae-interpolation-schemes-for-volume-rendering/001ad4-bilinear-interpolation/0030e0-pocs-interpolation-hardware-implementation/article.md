# POCS Interpolation Hardware Implementation
Also called: POCS Filter Hardware Implementation
## Overview
A hardware architecture for implementing Piecewise Constant (POCS) interpolation filters in volume rendering, designed to overcome the quality limitations of standard tri-linear interpolation while remaining feasible within VLSI constraints [S1, unofficial]. It distinguishes itself from [Bilinear Interpolation](858-bilinear-interpolation.md) by supporting larger kernel sizes (up to $5 \times 5 \times 5$) using lookup tables and specialized memory banking to handle the high data throughput required for multi-voxel sampling [S1, unofficial].
## How it works
### Inputs and Outputs
- **Input**: Volume dataset (voxel grid), ray sampling positions (fractional coordinates).
- **Output**: Interpolated scalar value for the sampling position.
- **Parameters**: Filter size ($3 \times 3 \times 3$ or $5 \times 5 \times 5$), RAM access time (assumed 100 ns in source), clock speed.

### Data Structures
1.  **RAM Banks**: Parallel memory modules storing voxel data.
    -   **Layout**: Voxel addresses are mapped to banks based on the parity (odd/even) of their row, column, and depth indices.
    -   **Bank Count**:
        -   For $3 \times 3 \times 3$ filter: 8 banks.
        -   For $5 \times 5 \times 5$ filter: 16 banks (derived from 64 voxels / 4x time multiplexing factor).
2.  **Lookup Table (LUT)**: Precomputed POCS function values stored in EPROM or similar static memory.
    -   **Content**: Interpolation weights or filter coefficients for specific fractional offsets.
    -   **Precision**: Determined by design requirements; values are precomputed to avoid runtime calculation overhead.
3.  **Interpolation Chip**: Contains the LUT, addressing logic, and caching registers.

### Numbered Steps
1.  **Address Parity Analysis**: For a given sampling point with fractional address $(x, y, z)$, determine the integer base address $(x_b, y_b, z_b)$ by truncation.
2.  **Bank Mapping**: Calculate the target RAM bank for each of the $N$ surrounding voxels required by the filter kernel.
    -   For each dimension $d \in \{x, y, z\}$, determine if the offset from the base address is odd or even.
    -   Map the combination of parities to a specific bank index $B \in \{0, \dots, 7\}$ for the 8-voxel case.
3.  **Parallel Fetch (8-voxel case)**: Issue simultaneous read commands to the 8 RAM banks.
    -   Ensure all 8 required voxels reside in distinct banks to avoid bank conflicts.
    -   Wait for data return (approx. 100 ns access time).
4.  **Time Multiplexing (64-voxel case)**:
    -   If using a $5 \times 5 \times 5$ filter (64 voxels), clock the RAM banks at 4x the system speed.
    -   Fetch voxels in groups of 4 over 4 clock cycles.
    -   This reduces the physical RAM bank count from 64 to 16, resulting in a 256-bit data path width.
5.  **Caching**: Store the fetched 64 voxels in the interpolation chip's internal cache.
    -   Exploit the fact that 4 resampling points typically fall within the spatial extent of a $3 \times 3 \times 3$ voxel sub-cube defined by the 64 surrounding voxels.
    -   Reuse the cached 64 voxels for all 4 resampling points to minimize memory bandwidth.
6.  **Lookup and Interpolation**:
    -   Use the fractional parts of the sampling coordinates as indices into the Lookup Table.
    -   Retrieve precomputed POCS weights.
    -   Multiply fetched voxel values by weights and sum to produce the output.

### Formulas
-   **Bank Assignment**: For a voxel at integer coordinates $(i, j, k)$, the bank index $B$ is determined by the parity bits:
    $$ B = 4 \cdot (i \bmod 2) + 2 \cdot (j \bmod 2) + 1 \cdot (k \bmod 2) $$
    *(Note: This mapping ensures that the 8 neighbors of any grid point map to unique banks 0-7.)*

-   **Data Path Width**:
    $$ W_{64} = \frac{64 \cdot w}{4} = 16 \cdot w $$
    Where $w$ is the voxel bit-width. For 16-bit voxels, $W_{64} = 256$ bits.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fetch_voxels_8_banks(base_x, base_y, base_z, ram_banks):
    """
    Fetches 8 surrounding voxels for a 3x3x3 filter.
    Assumes ram_banks is a list of 8 memory objects.
    """
    voxels = []
    # Iterate over all 8 neighbors (offsets -1, 0, 1)
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                # Calculate absolute address
                ax = base_x + dx
                ay = base_y + dy
                az = base_z + dz
                
                # Determine bank index based on parity of absolute address
                # Parity: 0 for even, 1 for odd
                px = ax % 2
                py = ay % 2
                pz = az % 2
                
                bank_idx = (px << 2) | (py << 1) | pz
                
                # Read from the specific bank
                voxels.append(ram_banks[bank_idx].read(ax, ay, az))
    return voxels

def fetch_voxels_16_banks_time_mux(base_x, base_y, base_z, ram_banks, clock_speed_factor=4):
    """
    Fetches 64 surrounding voxels for a 5x5x5 filter using time multiplexing.
    ram_banks has 16 entries.
    """
    all_voxels = []
    # 64 voxels total. 16 banks. 4 voxels per bank needed.
    # Clock banks 4x faster to fetch 4 times the data per cycle.
    
    for cycle in range(4):
        current_cycle_voxels = []
        for bank_idx in range(16):
            # Logic to determine which of the 4 voxels for this bank 
            # corresponds to the current cycle goes here.
            # This requires complex addressing logic to handle 
            # fractional address truncation shifts (see Pitfalls).
            voxel = ram_banks[bank_idx].read_fast(cycle)
            current_cycle_voxels.append(voxel)
        all_voxels.extend(current_cycle_voxels)
    return all_voxels

def interpolate_pocs(voxels, fractional_coords, lut):
    """
    Perform POCS interpolation using precomputed LUT.
    """
    # fractional_coords: (fx, fy, fz) in [0, 1)
    # lut: dictionary or array mapping fractional offsets to weights
    weight = lut.get(fractional_coords)
    
    # Simplified weighted sum (actual POCS may involve more complex basis functions)
    result = 0.0
    for i, v in enumerate(voxels):
        # In a real implementation, weights are derived from LUT based on 
        # relative position of voxel within the kernel.
        # Here we assume a simple average or pre-multiplied weight storage for brevity
        # as the source emphasizes the LUT for function values.
        result += v * weight[i] 
    return result
```

### Edge Cases and Pitfalls
-   **Address Truncation Mismatch**: When sampling two adjacent points $P_1$ and $P_2$, their truncated integer addresses may differ. If $P_1$'s truncated address maps to banks $\{A, B, C, D, E, F, G, H\}$, $P_2$'s truncated address might require a different subset of banks (e.g., shifting the parity requirements). Specifically, banks B, D, F, and H are sensitive to the parity of the coordinates. Extra addressing logic is required to detect if the truncated address is odd or even in any direction and adjust the bank selection accordingly [S1, unofficial].
-   **VLSI Pin Count**: A naive implementation of a $5 \times 5 \times 5$ filter would require 64 RAM banks and a 1024-bit data path, which is infeasible with current VLSI technology. Time multiplexing is mandatory [S1, unofficial].
-   **Memory Bandwidth**: The caching strategy relies on the assumption that 4 resampling points fall within the same $3 \times 3 \times 3$ sub-cube. If the ray step size is too large, this assumption fails, and the cache must be refilled more frequently, increasing bandwidth usage [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: $O(1)$ per sample point, dominated by memory access latency and LUT lookup.
-   **Memory Complexity**: $O(1)$ for the interpolation chip (LUT + cache). The RAM bank count scales with filter size and multiplexing factor.
-   **Performance**: The 8-voxel parallel fetch achieves throughput limited by RAM access time (100 ns). The 64-voxel implementation reduces hardware complexity (pins/banks) by 4x at the cost of requiring 4x RAM clock speed.
-   **Strengths**: Higher quality than tri-linear interpolation; feasible hardware implementation for large kernels.
-   **Limitations**: Requires significant extra addressing logic to handle bank conflicts when sampling points are close; high RAM clock speed requirements for large filters.

## Background
The method originates from research into Fourier volume rendering, specifically addressing the inadequacy of tri-linear interpolation for high-quality visualization [S1, unofficial]. It was proposed as a hardware-optimized alternative to software-based POCS filtering, leveraging lookup tables and memory banking to achieve real-time performance on VLSI hardware.

## Sources
[S1] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
