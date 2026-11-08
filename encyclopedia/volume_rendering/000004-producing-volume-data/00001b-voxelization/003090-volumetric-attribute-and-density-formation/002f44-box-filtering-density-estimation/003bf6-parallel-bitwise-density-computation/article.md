# Parallel Bitwise Density Computation

Also called: Bit-counting via bitwise operations, In-place z-density and xy-sum

## Overview
Parallel Bitwise Density Computation is a technique to compute voxel density (the number of filled voxels in a column) using parallel bitwise operations on integer textures, avoiding expensive texture lookups for bit counting [S1, unofficial]. It sets itself apart from sibling methods like Kernel Density Estimation by using exact integer arithmetic on binary slicemaps rather than probabilistic or smoothed field approximations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary slicemap (voxel grid) where each voxel is 1 bit (filled) or 0 bits (empty).
*   **Output:** A density map where each voxel stores the count of filled voxels in its column.

**Data Structures:**
*   **Integer Textures:** Voxels are packed into integer components. The method relies on the ability to perform bitwise operations on these integers in parallel.
*   **Bitmasks:** Specific integer patterns ($I_{mask}$) are used to isolate and align bit pairs.

**Steps:**

1.  **Compute Intermediate Z-Density Map:**
    Perform an in-place operation on the binary slicemap to store two bits per voxel. This encodes the pairwise sum of neighboring voxels in a column $c$. This step also performs downsampling. The operation is defined as:
    $$
    (c \& I_{10101\dots}) + ((c \& I_{01010\dots}) \gg 1)
    $$
    where:
    *   $\&$ is bitwise AND.
    *   $I_{10101\dots}$ is an integer with bitmask $10101\dots$ (isolating odd-positioned bits).
    *   $I_{01010\dots}$ is an integer with bitmask $01010\dots$ (isolating even-positioned bits).
    *   $\gg 1$ is a right shift by 1.
    *   The succeeding zero bit (introduced by the mask) after every copied bit ensures that the summation does not pollute the solution [S1, unofficial].

2.  **Sum Voxels in XY-Plane:**
    Sum up voxels of the z-density map in the xy-plane. Four neighboring voxels need to be combined. Since each voxel now represents 2 bits, their sum requires 4 bits of storage, preventing an in-place operation. To benefit from parallel execution, the sums of even and odd voxels are calculated separately:
    $$
    ESum := \sum_{i=0}^3 (c_i \& I_{00110011\dots}) \gg 2
    $$
    $$
    OSum := \sum_{i=0}^3 c_i \& I_{11001100\dots}
    $$
    where:
    *   $i$ iterates over the four neighbors in the xy-plane.
    *   $I_{00110011\dots}$ is a bitmask that recovers the right bit pairs and ensures each is followed by two zero bits for correct summation.
    *   $I_{11001100\dots}$ is a bitmask that recovers the left bit pairs and ensures each is followed by two zero bits for correct summation.
    *   $\gg 2$ is a right shift by 2.
    *   The resulting integers $ESum$ and $OSum$ are stored in the density map [S1, unofficial].

3.  **Result Ordering:**
    The voxels alternate along the z-direction between values in $EvenSum$ and $OddSum$. The order is $0,2/1,3$ rather than $0,1/2,3$. For normal derivation, this is not problematic, but for other applications, reordering may be required [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_bitwise_density(slicemap):
    """
    Computes density from a binary slicemap using parallel bitwise operations.
    slicemap: List of integers representing voxel columns.
    Returns: Density map with ESum and OSum interleaved.
    """
    # Define bitmasks
    # I_1010... isolates bits at positions 0, 2, 4... (assuming LSB is pos 0)
    # I_0101... isolates bits at positions 1, 3, 5...
    # Note: The exact bit pattern depends on the integer width and endianness.
    # For a 32-bit integer, I_1010... = 0xAAAAAAAA, I_0101... = 0x55555555
    I_1010 = 0xAAAAAAAA
    I_0101 = 0x55555555
    
    # Step 1: Compute Z-density (in-place)
    # (c & I_1010...) + ((c & I_0101...) >> 1)
    z_density = [(c & I_1010) + ((c & I_0101) >> 1) for c in slicemap]
    
    # Step 2: Sum in XY-plane
    # Assuming slicemap is organized such that neighbors are accessible.
    # For simplicity, assume we process groups of 4 neighbors.
    # c_i represents the z-density value of the i-th neighbor.
    
    # Masks for XY summation
    # I_00110011... isolates the lower 2 bits of each pair and shifts them to align
    # I_11001100... isolates the upper 2 bits of each pair
    # For 32-bit, I_00110011... = 0x33333333, I_11001100... = 0xCCCCCCCC
    I_0011 = 0x33333333
    I_1100 = 0xCCCCCCCC
    
    density_map = []
    for i in range(0, len(z_density), 4):
        # Get 4 neighbors
        c0, c1, c2, c3 = z_density[i], z_density[i+1], z_density[i+2], z_density[i+3]
        
        # Calculate ESum and OSum
        # ESum: sum of lower 2 bits of each neighbor, shifted right by 2
        esum = ((c0 & I_0011) >> 2) + ((c1 & I_0011) >> 2) + ((c2 & I_0011) >> 2) + ((c3 & I_0011) >> 2)
        
        # OSum: sum of upper 2 bits of each neighbor
        osum = (c0 & I_1100) + (c1 & I_1100) + (c2 & I_1100) + (c3 & I_1100)
        
        # Store in density map
        density_map.append(esum)
        density_map.append(osum)
        
    return density_map
```

**Parameters:**
*   **Bitmask Width:** Depends on the integer width of the texture components (e.g., 8-bit, 16-bit, 32-bit). Typical values are 8-bit or 16-bit per component [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on exact integer arithmetic. Overflow must be avoided by ensuring sufficient bit width for the sums [S1, unofficial].
*   **Boundaries:** The method assumes a regular grid structure. Boundary conditions must be handled appropriately [S1, unofficial].
*   **Ordering:** The resulting density map has an alternating order ($0,2/1,3$). This may require reordering for certain applications [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels. The bitwise operations are constant time per voxel.
*   **Memory Complexity:** $O(N)$ for the input and output density maps.
*   **Performance:** The method is highly efficient on GPUs, especially with DX10+ hardware that supports arithmetic bitwise operations. DX9 implementations may use lookup textures, which can be slower [S1, unofficial].
*   **Strengths:** Exact density computation, no texture lookups required (on modern hardware), parallelizable.
*   **Limitations:** Requires integer textures, ordering of results may be non-intuitive, limited by integer bit width.

## Background
Parallel Bitwise Density Computation was introduced in the context of single-pass GPU solid voxelization for real-time applications [S1, unofficial]. It was developed to address the inefficiency of bit counting with textures, particularly for translucency effects and CSG operations [S1, unofficial]. The method leverages the parallel nature of GPU arithmetic to perform bit counting in a single pass [S1, unofficial].

## Sources
[S1] Single-Pass GPU Solid Voxelization for Real-Time Applications, https://wo1fsea.github.io/files/About_NavMesh/P3_Single-Pass_GPU_Solid_Voxelization_for_Real-Time_Applications.pdf, fetched 2026-10-29 (unofficial)
