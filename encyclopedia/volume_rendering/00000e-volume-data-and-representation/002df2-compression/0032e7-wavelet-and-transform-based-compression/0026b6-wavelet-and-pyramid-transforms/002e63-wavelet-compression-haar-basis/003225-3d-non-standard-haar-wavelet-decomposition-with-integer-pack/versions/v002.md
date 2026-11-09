# 3D Non-standard Haar Wavelet Decomposition with Integer Packing

Also called: Haar wavelet decomposition, Integer-based wavelet storage

## Overview
This method performs a 3D Haar wavelet decomposition on volume data using unsigned 32-bit integers to store coefficients, replacing floating-point arithmetic to leverage faster integer operations without loss of precision [S1]. It differs from standard float-based wavelet compression by utilizing the unused bits of a 32-bit integer word to handle intermediate overflows and underflows, allowing for a fixed-point hierarchy that occupies the same memory footprint as the original volume [S1].

## How it works
**Inputs:** A 3D volume array of scalar values with 8-bit resolution (0–255 gray levels) [S1].
**Outputs:** A multiresolution hierarchy of wavelet coefficients stored in unsigned 32-bit integers [S1].

**Data Structures:**
*   **Coefficient Storage:** Unsigned 32-bit integer (`uint32`) [S1].
*   **Bit Layout:**
    *   Bits 0–22: Available for overflow/underflow handling during iterative decomposition [S1].
    *   Bits 23–30: Initial storage for the 8-bit input data [S1].
    *   Bit 31: Unused initially; used for overflow from sum operations [S1].

**Parameters:**
*   **Threshold:** A scalar value used to determine if a detail coefficient is negligible and can be set to zero for compression [S1].
*   **Precision Allocation:** Low-frequency coefficients may be stored at higher precision than high-frequency coefficients [S1].

**Algorithm Steps:**
1.  **Initialization:** Load the 8-bit input voxel values into bits 23–30 of a 32-bit unsigned integer, ensuring bits 0–22 and 31 are zeroed [S1].
2.  **Iterative Decomposition:** Repeat the following cycle for each dimension (x, y, z) until the desired resolution or bit limit is reached [S1]:
    a.  **Pairing:** Group adjacent voxels along the current dimension [S1].
    b.  **Approximation (Low-pass):** Compute the sum of the two paired values [S1]. If the sum exceeds 255, it overflows into bit 31 [S1].
    c.  **Shift (Downsample):** Perform a right bit shift by 1 (division by 2) on the sum to obtain the reduced pixel value (approximation coefficient) [S1].
    d.  **Detail (High-pass):** Compute the detail coefficient using the same division-by-two mechanism [S1].
    e.  **Storage:** Store the resulting approximation and detail coefficients back into the integer structure, utilizing bits 0–22 to absorb the 3-bit underflow potential across the three dimensions [S1].
3.  **Termination:** The algorithm can be applied 7 times before exhausting the 22 available bits for underflow/overflow handling [S1]. After 7 iterations, the data is reduced to $(1/8)^7$ of the original size [S1].

**Pseudocode:**
```python
def decompose_volume(volume_3d):
    """
    Performs 3D non-standard Haar wavelet decomposition using integer packing.
    volume_3d: List of lists of lists of integers (0-255).
    Returns: Approximation grid at level 7 and a list of detail coefficient grids.
    """
    # Initialize coefficients with 8-bit data in bits 23-30
    current = [[[ (v << 23) for v in row] for row in slice] for slice in volume_3d]
    details = []
    
    # Perform 7 levels of decomposition
    for level in range(7):
        level_details = []
        
        # Pass 1: X-dimension
        next_x = []
        for y in range(len(current)):
            row_next = []
            for z in range(len(current[0])):
                col_next = []
                for x in range(0, len(current[y]), 2):
                    a = current[y][z][x]
                    b = current[y][z][x+1]
                    approx = (a + b) >> 1
                    detail = (a - b) >> 1
                    col_next.append(approx)
                    level_details.append(detail)
                row_next.append(col_next)
            next_x.append(row_next)
        current = next_x
        
        # Pass 2: Y-dimension
        next_y = []
        for z in range(len(current[0][0])):
            row_next = []
            for x in range(len(current)):
                col_next = []
                for y in range(0, len(current[x]), 2):
                    a = current[y][z][x]
                    b = current[y+1][z][x]
                    approx = (a + b) >> 1
                    detail = (a - b) >> 1
                    col_next.append(approx)
                    level_details.append(detail)
                row_next.append(col_next)
            next_y.append(row_next)
        current = next_y
        
        # Pass 3: Z-dimension
        next_z = []
        for y in range(len(current[0])):
            row_next = []
            for x in range(len(current)):
                col_next = []
                for z in range(0, len(current[0][0]), 2):
                    a = current[y][z][x]
                    b = current[y][z+1][x]
                    approx = (a + b) >> 1
                    detail = (a - b) >> 1
                    col_next.append(approx)
                    level_details.append(detail)
                row_next.append(col_next)
            next_z.append(row_next)
        current = next_z
        
        details.append(level_details)
        
    return current, details

**Edge Cases and Pitfalls:**
*   **Bit Exhaustion:** The algorithm can only be applied 7 times because 22 bits are available to handle the 3-bit underflow per dimension cycle [S1].
*   **Overflow:** Intermediate sums can exceed 255, overflowing into bit 31. This bit must be preserved during shifts [S1].
*   **Underflow:** Right shifts cause loss of precision. The 22 lower bits absorb this loss [S1].
*   **Thresholding:** A threshold must be applied to detail coefficients to achieve compression; coefficients below the threshold are set to zero [S1].

## Operating characteristics
*   **Memory Complexity:** The wavelet hierarchy uses the same amount of memory as the original volume [S1].
*   **Time Complexity:** Each level reduces the data size by a factor of 8, leading to exponential reduction per level [S1].
*   **Performance:** Integer arithmetic is significantly faster than floating-point arithmetic [S1].
*   **Strengths:** No loss of precision in the decomposition process itself (until thresholding); fast integer operations; fixed memory footprint [S1].
*   **Limitations:** Limited to 7 levels of decomposition due to bit constraints; requires careful handling of overflow/underflow bits [S1].

## Background
This method was developed for giga-scale multiresolution volume rendering on distributed display clusters [S1]. It was implemented on a cluster of 25 PowerMac G5 computers with nVidia GeForce 6800 and Quadro FX4500 graphics cards driving 50 Apple Cinema Displays [S1]. The approach leverages the fact that wavelet transforms allow storing the hierarchy in the same memory as the original data, and replacing floats with integers accelerates computation [S1].

## Sources
[S1] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
