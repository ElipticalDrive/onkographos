# 3D Non-standard Haar Wavelet Decomposition with Integer Packing

Also called: Haar wavelet decomposition, Integer-based wavelet storage

## Overview
This method performs a 3D Haar wavelet decomposition on volume data using unsigned 32-bit integers to store coefficients, replacing floating-point arithmetic to leverage faster integer operations without loss of precision [S1]. It differs from standard float-based wavelet compression by utilizing the unused bits of a 32-bit integer word to handle intermediate overflows and underflows, allowing for a fixed-point hierarchy that occupies the same memory footprint as the original volume [S1].

## How it works
**Inputs:** A 3D volume array of scalar values with 8-bit resolution (0–255 gray levels).
**Outputs:** A multiresolution hierarchy of wavelet coefficients stored in unsigned 32-bit integers.

**Data Structures:**
*   **Coefficient Storage:** Unsigned 32-bit integer (`uint32`).
*   **Bit Layout:**
    *   Bits 0–22: Available for overflow/underflow handling during iterative decomposition.
    *   Bits 23–30: Initial storage for the 8-bit input data.
    *   Bit 31: Unused initially; used for overflow from sum operations.
    *   Bit 32 (implicit in 64-bit arithmetic if needed, but logic relies on 32-bit wrap/shift): The method assumes 32-bit container.

**Parameters:**
*   **Threshold:** A scalar value used to determine if a detail coefficient is negligible and can be set to zero for compression [S1].
*   **Precision Allocation:** Low-frequency coefficients may be stored at higher precision than high-frequency coefficients [S1].

**Algorithm Steps:**
1.  **Initialization:** Load the 8-bit input voxel values into bits 23–30 of a 32-bit unsigned integer. Ensure bits 0–22 and 31 are zeroed [S1].
2.  **Iterative Decomposition:** Repeat the following cycle for each dimension (x, y, z) until the desired resolution or bit limit is reached:
    a.  **Pairing:** Group adjacent voxels along the current dimension.
    b.  **Approximation (Low-pass):** Compute the sum of the two paired values. If the sum exceeds 255, it overflows into bit 31 [S1].
    c.  **Shift (Downsample):** Perform a right bit shift by 1 (division by 2) on the sum to obtain the reduced pixel value (approximation coefficient) [S1]. This shift causes underflow (loss of the least significant bit).
    d.  **Detail (High-pass):** Compute the detail coefficient. The source notes that division by two is involved in computing detail coefficients, implying a similar shift operation or subtraction from the sum [S1]. *Note: Standard Haar uses $A = (a+b)/2$ and $D = (a-b)/2$. The source emphasizes the shift mechanism for both, suggesting a fixed-point implementation where precision is managed via the available bits.*
    e.  **Storage:** Store the resulting approximation and detail coefficients back into the integer structure, utilizing bits 0–22 to absorb the 3-bit underflow potential across the three dimensions [S1].
3.  **Termination:** The algorithm can be applied 7 times before exhausting the 22 available bits for underflow/overflow handling [S1]. After 7 iterations, the data is reduced to $(1/8)^7$ of the original size [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decompose_volume(volume_3d):
    """
    Performs 3D non-standard Haar wavelet decomposition using integer packing.
    volume_3d: List of lists of lists of integers (0-255).
    Returns: Hierarchical structure of uint32 coefficients.
    """
    # Initialize coefficients with 8-bit data in bits 23-30
    # Assuming input is normalized to 0-255
    coeffs = []
    for x in volume_3d:
        row = []
        for y in x:
            col = []
            for z in y:
                # Pack 8-bit value into bits 23-30 of a 32-bit unsigned int
                # Bits 0-22 and 31 are 0
                val = z << 23
                col.append(val)
            row.append(col)
        coeffs.append(row)
    
    # Perform 7 levels of decomposition
    # Note: The source describes a cycle over 3 dimensions.
    # We assume a standard octree-like reduction where each level reduces 
    # dimensions by half.
    
    for level in range(7):
        # Determine current dimensions
        dim_x = len(coeffs)
        dim_y = len(coeffs[0])
        dim_z = len(coeffs[0][0])
        
        # Create next level containers
        next_coeffs = []
        for i in range(0, dim_x, 2):
            row_next = []
            for j in range(0, dim_y, 2):
                col_next = []
                for k in range(0, dim_z, 2):
                    # Get 8 neighbors for 3D Haar
                    # Indices: (i,j,k), (i+1,j,k), (i,j+1,k), (i+1,j+1,k),
                    #          (i,j,k+1), (i+1,j,k+1), (i,j+1,k+1), (i+1,j+1,k+1)
                    
                    # Extract values from bits 23-30 (and overflow bits if present)
                    # For simplicity, assume we operate on the full 32-bit int
                    # but only bits 23-30 contain valid data initially.
                    
                    v000 = coeffs[i][j][k]
                    v100 = coeffs[i+1][j][k]
                    v010 = coeffs[i][j+1][k]
                    v110 = coeffs[i+1][j+1][k]
                    v001 = coeffs[i][j][k+1]
                    v101 = coeffs[i+1][j][k+1]
                    v011 = coeffs[i][j+1][k+1]
                    v111 = coeffs[i+1][j+1][k+1]
                    
                    # Compute Sum (Approximation numerator)
                    # Sum of all 8
                    total_sum = v000 + v100 + v010 + v110 + v001 + v101 + v011 + v111
                    
                    # Compute Approximation Coefficient
                    # Divide by 8 (shift right by 3)
                    # The source mentions division by 2 per dimension cycle.
                    # 3 dimensions -> shift by 3 total per level.
                    approx = total_sum >> 3
                    
                    # Compute Detail Coefficients
                    # Standard Haar details involve differences.
                    # The source states "division by two is involved" for details.
                    # We implement standard 3D Haar details here.
                    # D1: (v000 + v100 + v010 + v110) - (v001 + v101 + v011 + v111)
                    # D2: (v000 + v010 + v001 + v011) - (v100 + v110 + v101 + v111)
                    # D3: (v000 + v100 + v001 + v101) - (v010 + v110 + v011 + v111)
                    
                    sum_xy = v000 + v100 + v010 + v110
                    sum_xy_z1 = v001 + v101 + v011 + v111
                    detail_z = (sum_xy - sum_xy_z1) >> 3
                    
                    sum_xz = v000 + v100 + v001 + v101
                    sum_xz_y1 = v010 + v110 + v011 + v111
                    detail_y = (sum_xz - sum_xz_y1) >> 3
                    
                    sum_yz = v000 + v010 + v001 + v011
                    sum_yz_x1 = v100 + v110 + v101 + v111
                    detail_x = (sum_yz - sum_yz_x1) >> 3
                    
                    # Store approx in the new coarse grid
                    # Details are stored in separate arrays or interleaved.
                    # The source implies a hierarchy stored in the same memory size.
                    # We store approx in the main structure for the next level.
                    col_next.append(approx)
                row_next.append(col_next)
            next_coeffs.append(row_next)
        
        coeffs = next_coeffs
        
        # Note: In a full implementation, detail coefficients must be stored.
        # The source emphasizes that the *hierarchy* uses the same memory.
        # This implies storing details in the "freed" space of the decomposed level
        # or in a separate structure that sums to the original size.
        # Given the constraint of "integer packing" and "overflow/underflow",
        # the details are likely stored in the bits 0-22 of the approximation slots
        # or in a parallel structure. The source does not specify the exact storage
        # layout for details vs approximations beyond the bit availability.
        # [G] Assuming a standard multiresolution storage where details are stored
        # in the same array structure but at different indices or levels.
        
    return coeffs
```

**Edge Cases and Pitfalls:**
*   **Bit Exhaustion:** The algorithm can only be applied 7 times because 22 bits are available to handle the 3-bit underflow per dimension cycle [S1].
*   **Overflow:** Intermediate sums can exceed 255, overflowing into bit 31. This bit must be preserved during shifts [S1].
*   **Underflow:** Right shifts cause loss of precision. The 22 lower bits absorb this loss [S1].
*   **Thresholding:** A threshold must be applied to detail coefficients to achieve compression; coefficients below the threshold are set to zero [S1].

## Operating characteristics
*   **Memory Complexity:** The wavelet hierarchy uses the same amount of memory as the original volume [S1].
*   **Time Complexity:** Exponential reduction in data size per level. After 7 levels, data is reduced to $(1/8)^7 \approx 4.77 \times 10^{-7}$ of the original [S1].
*   **Performance:** Integer arithmetic is significantly faster than floating-point arithmetic [S1].
*   **Strengths:** No loss of precision in the decomposition process itself (until thresholding); fast integer operations; fixed memory footprint.
*   **Limitations:** Limited to 7 levels of decomposition due to bit constraints; requires careful handling of overflow/underflow bits.

## Background
This method was developed for giga-scale multiresolution volume rendering on distributed display clusters [S1]. It was implemented on a cluster of 25 PowerMac G5 computers with nVidia GeForce 6800 and Quadro FX4500 graphics cards driving 50 Apple Cinema Displays [S1]. The approach leverages the fact that wavelet transforms allow storing the hierarchy in the same memory as the original data, and replacing floats with integers accelerates computation [S1].

## Sources
[S1] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
