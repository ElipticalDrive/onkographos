# POCS Interpolation Hardware Implementation
Also called: POCS Filter Hardware Implementation
## Overview
A hardware architecture for implementing POCS interpolation filters in volume rendering, designed to overcome the quality limitations of standard tri-linear interpolation while remaining feasible within VLSI constraints [S1, unofficial]. It distinguishes itself from [Bilinear Interpolation](858-bilinear-interpolation.md) by supporting larger kernel sizes ($3 \times 3 \times 3$ and $5 \times 5 \times 5$) using lookup tables and specialized memory banking to handle the high data throughput required for multi-voxel sampling [S1, unofficial].
## How it works
### Inputs and Outputs
- **Input**: Volume dataset (voxel grid), ray sampling positions (fractional coordinates) [S1, unofficial].
- **Output**: Interpolated scalar value for the sampling position [S1, unofficial].
- **Parameters**: Filter size ($3 \times 3 \times 3$ or $5 \times 5 \times 5$), RAM access time (assumed 100 ns in source), clock speed [S1, unofficial].

### Data Structures
1. **RAM Banks**: Parallel memory modules storing voxel data [S1, unofficial].
    - **Layout**: Voxel addresses are mapped to banks based on the parity (odd/even) of their row, column, and depth indices [S1, unofficial].
    - **Bank Count**: 8 banks for the $3 \times 3 \times 3$ filter, and 16 banks for the $5 \times 5 \times 5$ filter (derived from 64 voxels divided by a 4x time multiplexing factor) [S1, unofficial].
2. **Lookup Table (LUT)**: Precomputed POCS function values stored in EPROM or similar static memory [S1, unofficial].
    - **Content**: Interpolation weights or filter coefficients for specific fractional offsets [S1, unofficial].
    - **Precision**: Determined by design requirements; values are precomputed to avoid runtime calculation overhead [S1, unofficial].
3. **Interpolation Chip**: Contains the LUT, addressing logic, and caching registers [S1, unofficial].

### Numbered Steps
1. **Address Parity Analysis**: For a given sampling point with fractional address $(x, y, z)$, determine the integer base address $(x_b, y_b, z_b)$ by truncation [S1, unofficial].
2. **Bank Mapping**: Calculate the target RAM bank for each of the $N$ surrounding voxels required by the filter kernel [S1, unofficial].
    - For each dimension $d \in \{x, y, z\}$, determine if the offset from the base address is odd or even [S1, unofficial].
    - Map the combination of parities to a specific bank index $B \in \{0, \dots, 7\}$ for the 8-voxel case [S1, unofficial].
3. **Parallel Fetch (8-voxel case)**: Issue simultaneous read commands to the 8 RAM banks [S1, unofficial].
    - Ensure all 8 required voxels reside in distinct banks to avoid bank conflicts [S1, unofficial].
    - Wait for data return (approx. 100 ns access time) [S1, unofficial].
4. **Time Multiplexing (64-voxel case)**:
    - If using a $5 \times 5 \times 5$ filter (64 voxels), clock the RAM banks at 4x the system speed [S1, unofficial].
    - Fetch voxels in groups of 4 over 4 clock cycles [S1, unofficial].
    - This reduces the physical RAM bank count from 64 to 16, resulting in a 256-bit data path width [S1, unofficial].
5. **Caching**: Store the fetched 64 voxels in the interpolation chip's internal cache [S1, unofficial].
    - Exploit the fact that 4 resampling points typically fall within the spatial extent of a voxel sub-cube defined by the 64 surrounding voxels [S1, unofficial].
    - Reuse the cached 64 voxels for all 4 resampling points to minimize memory bandwidth [S1, unofficial].
6. **Lookup and Interpolation**:
    - Use the fractional parts of the sampling coordinates as indices into the Lookup Table [S1, unofficial].
    - Retrieve precomputed POCS weights [S1, unofficial].
    - Multiply fetched voxel values by weights and sum to produce the output [S1, unofficial].

### Formulas
- **Bank Assignment**: For a voxel at integer coordinates $(i, j, k)$, the bank index $B$ is determined by the parity bits:
  $$ B = 4 \cdot (i \bmod 2) + 2 \cdot (j \bmod 2) + 1 \cdot (k \bmod 2) $$
  *(Note: This mapping ensures that the 8 neighbors of any grid point map to unique banks 0-7) [S1, unofficial].*

- **Data Path Width**:
  $$ W_{64} = \frac{64 \cdot w}{4} = 16 \cdot w $$
  Where $w$ is the voxel bit-width. For 16-bit voxels, $W_{64} = 256$ bits [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fetch_voxels_8_banks(base_x, base_y, base_z, ram_banks):
    """Fetches 8 surrounding voxels for a 3x3x3 filter using 8 parallel RAM banks."""
    voxels = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                ax, ay, az = base_x + dx, base_y + dy, base_z + dz
                bank_idx = ((ax % 2) << 2) | ((ay % 2) << 1) | (az % 2)
                voxels.append(ram_banks[bank_idx][ax, ay, az])
    return voxels

def fetch_voxels_16_banks_time_mux(base_x, base_y, base_z, ram_banks):
    """Fetches 64 surrounding voxels for a 5x5x5 filter using 16 RAM banks clocked 4x faster."""
    all_voxels = []
    # 64 voxels total, 16 banks, 4 voxels per bank fetched over 4 cycles
    for cycle in range(4):
        for bank_idx in range(16):
            # The exact cycle-to-voxel mapping logic is not specified in the sources.
            # In practice, extra addressing logic distinguishes cases based on 
            # whether the truncated address is odd or even in any direction.
            voxel = ram_banks[bank_idx].read_fast(cycle)
            all_voxels.append(voxel)
    return all_voxels

def interpolate_pocs(voxels, fractional_coords, lut):
    """Perform POCS interpolation using precomputed LUT."""
    weight = lut.get(fractional_coords)
    result = 0.0
    for i, v in enumerate(voxels):
        result += v * weight[i]
    return result
```

### Edge Cases and Pitfalls
- **Address Truncation Mismatch**: When sampling two points $P_1$ and $P_2$ with fractional addresses, the truncated address of $P_2$ may not map to the same 8 RAM banks as $P_1$ (e.g., banks B, D, F, H might differ) [S1, unofficial]. Extra addressing logic is needed to distinguish cases based on whether the truncated address is odd or even in any direction [S1, unofficial].
- **VLSI Pin Count**: A naive implementation of a $5 \times 5 \times 5$ filter would require 64 RAM banks and a 1024-bit data path, which is infeasible with current VLSI technology [S1, unofficial]. Time multiplexing is used to reduce the RAM bank count by a factor of 4, resulting in a 256-bit data path [S1, unofficial].
- **Memory Bandwidth**: Caching voxels in the interpolation chip exploits that 4 resampling points exist within one voxel sub-cube of 8 voxels, allowing reuse of the same 64 voxels [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(1)$ per sample point, dominated by memory access latency and LUT lookup [S1, unofficial].
- **Memory Complexity**: $O(1)$ for the interpolation chip (LUT + cache). The RAM bank count scales with filter size and multiplexing factor [S1, unofficial].
- **Performance**: The 8-voxel parallel fetch achieves throughput limited by RAM access time (100 ns) [S1, unofficial]. The 64-voxel implementation reduces hardware complexity (pins/banks) by 4x at the cost of requiring 4x RAM clock speed [S1, unofficial].
- **Strengths**: Higher quality than tri-linear interpolation; feasible hardware implementation for large kernels [S1, unofficial].
- **Limitations**: Requires significant extra addressing logic to handle bank conflicts when sampling points are close; high RAM clock speed requirements for large filters [S1, unofficial].

## Background
The method originates from research on hardware implementations of POCS filters for volume rendering, specifically addressing the inadequacy of tri-linear interpolation for high-quality visualization [S1, unofficial]. It was proposed as a hardware-optimized alternative to software-based POCS filtering, leveraging lookup tables and memory banking to achieve real-time performance on VLSI hardware [S1, unofficial].

## Sources
[S1] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
