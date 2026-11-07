# Adaptive Thresholding for Rapid Transfer Function Switching

Also called: Density encoding scheme, Two-bit cell coding

## Overview
Adaptive Thresholding for Rapid Transfer Function Switching is a classification optimization that enables instant switching between a fixed set of predefined transfer functions without re-evaluating voxel data or re-classifying the volume [S1, unofficial]. Unlike standard pre-classification which requires re-computation when opacity thresholds change, this method pre-computes a two-bit code per voxel cell based on density intervals, allowing the renderer to skip transparent cells by simply changing a lookup table pointer [S1, unofficial]. It is distinct from general binary classification [6229-binary-and-soft-classification.md] by its specific use of multi-bit interval coding to support multiple simultaneous threshold configurations.

## How it works
**Inputs:**
*   A scalar volume dataset $V$ with dimensions $D_x \times D_y \times D_z$.
*   A set of $N$ predefined transfer functions, each defined by a lower density threshold $T_i$.
*   A rendering ray traversal routine (e.g., Depth routine).

**Outputs:**
*   A pre-computed two-bit code array $C$ of size $D_x \times D_y \times D_z$.
*   A set of Lookup Tables (LUTs) for rapid cell skipping.

**Data Structures:**
1.  **Code Array ($C$):** An integer array where each element stores a 2-bit code for a corresponding voxel cell. The array is packed such that codes are contiguous but must be treated as atomic 2-bit units during memory operations.
2.  **Lookup Table (LUT):** A small table mapping byte patterns to bit offsets. For a specific transfer function, the LUT contains the offset of the first occurrence of the required non-transparent bit pattern (e.g., '10' or '11') within a byte.

**Steps:**

1.  **Define Intervals:**
    Let the $N$ transfer functions have lower density thresholds $T_0 < T_1 < \dots < T_{N-1}$. These thresholds divide the density domain into $N+1$ intervals $I_0, I_1, \dots, I_N$, sorted in ascending order.
    *   $I_0$: $[min\_density, T_0)$
    *   $I_k$: $[T_{k-1}, T_k)$ for $0 < k < N$
    *   $I_N$: $[T_{N-1}, max\_density]$

2.  **Assign Codes:**
    Assign a unique 2-bit binary code to each interval index $k$. The code is the binary representation of $k$ using 2 bits.
    *   Code for $I_0$: `00`
    *   Code for $I_1$: `01`
    *   Code for $I_2$: `10`
    *   Code for $I_3$: `11`
    *(Note: This supports up to 4 intervals, i.e., 3 transfer functions, as per the source example.)*

3.  **Compute Cell Codes:**
    For each voxel cell (or voxel, depending on implementation granularity) at position $\mathbf{x}$:
    *   Identify the density value $v$ of the "highest corner" voxel. In the context of the source, this refers to the voxel value determining the cell's classification status. Let $v = V[\mathbf{x}]$.
    *   Determine which interval $I_k$ contains $v$.
    *   Store the 2-bit code for $k$ in $C[\mathbf{x}]$.

4.  **Store and Shear:**
    Store $C$ in memory. When applying shear transformations (e.g., for rotation), the two bits of each code must be moved together as a unit to avoid splitting the code across memory boundaries or misalignment [S1, unofficial].

5.  **Rendering (LUT Selection):**
    *   When a user selects a transfer function $TF_j$ with threshold $T_j$, identify the required non-transparent codes. Samples with density $< T_j$ are transparent (codes $0$ to $j-1$). Samples with density $\ge T_j$ are potentially opaque (codes $j$ to $N$).
    *   The renderer searches for the first cell with a code $\ge j$.
    *   The Depth routine uses a LUT specific to $TF_j$. The LUT maps byte patterns to the bit offset of the first matching pattern (e.g., searching for '10' or '11' if $j=2$).
    *   Update the LUT pointer to the address of the LUT corresponding to $TF_j$. This allows the renderer to skip transparent cells rapidly without re-evaluating the transfer function for every sample [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def precompute_codes(volume, thresholds):
    """
    Pre-compute two-bit codes for rapid TF switching.
    
    Args:
        volume: 3D array of voxel densities.
        thresholds: List of N lower density thresholds [T0, T1, ..., T_{N-1}].
                    Sorted ascending.
    
    Returns:
        codes: 2D/3D array of 2-bit integers.
        intervals: List of interval boundaries for reference.
    """
    # Define interval boundaries
    # I_0: [min, T0), I_1: [T0, T1), ..., I_N: [T_{N-1}, max]
    # Number of intervals is len(thresholds) + 1
    num_intervals = len(thresholds) + 1
    
    # Ensure codes fit in 2 bits (max 4 intervals)
    if num_intervals > 4:
        raise ValueError("Adaptive Thresholding supports up to 4 intervals (3 thresholds) with 2-bit coding.")

    min_val = volume.min()
    max_val = volume.max()
    
    # Initialize codes array
    codes = np.zeros_like(volume, dtype=np.uint8)
    
    for x in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for z in range(volume.shape[2]):
                v = volume[x, y, z]
                
                # Determine interval index k
                k = 0
                for i, t in enumerate(thresholds):
                    if v >= t:
                        k = i + 1
                    else:
                        break
                
                # Store 2-bit code
                # k is 0, 1, 2, or 3
                codes[x, y, z] = k
                
    return codes

def get_lut_for_tf(threshold_index, num_intervals):
    """
    Generate a Lookup Table for skipping transparent cells.
    
    Args:
        threshold_index: The index of the selected transfer function's threshold.
                         If TF_j is selected, we skip codes 0 to j-1.
                         We search for codes >= threshold_index.
        num_intervals: Total number of intervals (max 4).
    
    Returns:
        lut: Dictionary or array mapping byte patterns to bit offsets.
             Simplified: returns the minimum code value to search for.
    """
    # The LUT helps find the first byte containing a code >= threshold_index.
    # For implementation simplicity in this context, the LUT effectively
    # tells the renderer which bit patterns are "active".
    # In the source, the LUT contains bit offsets of the first matching pattern.
    # Here we return the target code range.
    return threshold_index

def render_with_rapid_switching(codes, ray, current_tf_index, lut):
    """
    Render a single ray using the pre-computed codes and LUT.
    
    Args:
        codes: Pre-computed 2-bit code array.
        ray: Ray object with origin and direction.
        current_tf_index: Index of the currently selected transfer function.
        lut: Lookup table for the current TF.
    
    Returns:
        Color and opacity of the first non-transparent hit.
    """
    # Intersect ray with volume
    t_min, t_max = ray.intersect_volume()
    
    # Sample along ray
    t = t_min
    while t < t_max:
        # Get voxel coordinates
        x, y, z = ray.get_coordinates(t)
        
        # Get code for this cell
        code = codes[x, y, z]
        
        # Check if code is in the non-transparent range for current TF
        # Codes < current_tf_index are transparent (skipped)
        if code >= current_tf_index:
            # Found a potentially opaque cell
            # Perform standard volume rendering step (e.g., alpha blending)
            # using the actual transfer function for this TF
            color, alpha = evaluate_tf_and_sample(volume, ray, t, current_tf_index)
            return color, alpha
        
        # Advance ray
        t += ray.step_size
        
    return 0.0, 0.0
```

**Parameters:**
*   **Number of Transfer Functions ($N$):** The source example uses $N=3$, creating 4 intervals ($I_0$ to $I_3$) and 2-bit codes [S1, unofficial]. This limits the method to 4 distinct density regions.
*   **Cell Size:** The method can be applied per voxel or per cell. The source reports performance for cell sizes 1, 2, and 4 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Bit Alignment:** When shearing or transforming the code array, the two bits of a code must be moved together. Splitting a code across memory boundaries or misaligning bits will corrupt the classification data [S1, unofficial].
*   **Interval Limit:** The 2-bit encoding restricts the number of transfer functions to 4 (indices 0-3). More than 3 thresholds cannot be supported with this specific coding scheme [S1, unofficial].
*   **Threshold Sorting:** Thresholds must be strictly sorted in ascending order to ensure correct interval assignment [S1, unofficial].

## Subtopics
*   **Binary Classification [6229-binary-and-soft-classification.md]:** Uses hard thresholds for inclusion/exclusion. Adaptive Thresholding extends this by pre-computing interval codes to support multiple binary-like switches without re-classification.
*   **Pre-classification [6103-pre-classification.md]:** Applies transfer functions before sampling. Adaptive Thresholding is a form of pre-classification that optimizes the lookup process for multiple TFs.
*   **1D Lookup Table with Interpolation [2528-1d-lookup-table-with-interpolation.md]:** Standard method for evaluating TFs. Adaptive Thresholding replaces the per-sample TF evaluation with a code-based skip mechanism.

## Operating characteristics
*   **Time Complexity:** Pre-computation is $O(V)$ where $V$ is the number of voxels. Rendering time per ray is reduced because transparent cells are skipped using the LUT, avoiding per-sample TF evaluation for transparent regions.
*   **Memory Complexity:** Requires an additional integer array of size $V$ storing 2 bits per voxel. This is minimal overhead compared to the volume data itself.
*   **Measured Performance:**
    *   For a $128 \times 128 \times 113$ volume with low opacity TFs:
        *   Cell size 1: TF A takes 0.36s, TF B 0.19s, TF C 0.18s [S1, unofficial].
        *   Cell size 2: TF A 0.39s, TF B 0.21s, TF C 0.21s [S1, unofficial].
        *   Cell size 4: TF A 0.51s, TF B 0.32s, TF C 0.31s [S1, unofficial].
    *   For a $256 \times 256 \times 225$ volume with low opacity TFs:
        *   Cell size 1: TF A 1.97s, TF B 1.02s, TF C 0.99s [S1, unofficial].
        *   Cell size 2: TF A 1.93s, TF B 1.01s, TF C 0.96s [S1, unofficial].
        *   Cell size 4: TF A 2.44s, TF B 1.45s, TF C 1.46s [S1, unofficial].
    *   Performance is largely independent of the selected transfer function for skipping, as the LUT pointer change is $O(1)$ [S1, unofficial].
*   **Strengths:** Rapid switching between predefined TFs; no re-classification needed; efficient skipping of transparent cells.
*   **Limitations:** Limited to 4 intervals (3 thresholds); requires pre-computation; less flexible than dynamic TF evaluation for arbitrary thresholds.
*   **When to Use:** When a small, fixed set of transfer functions is used frequently (e.g., medical imaging with standard tissue types like air, fat, soft tissue, bone) and rapid switching is required [S1, unofficial].
*   **When Not to Use:** When the number of transfer functions exceeds 4, or when thresholds are dynamic/user-defined in real-time.

## Background
The method was proposed in the context of fast volume rotation and rendering techniques for medical data visualization [S1, unofficial]. It addresses the limitation of pre-classified binary volumes where the opacity function cannot be modified flexibly [S1, unofficial]. The technique is particularly relevant in medical imaging where densities can be separated into a few distinct materials (e.g., air, fat, soft tissue, bone) based on Hounsfield units [S1, unofficial]. It was implemented in C++ and tested on an SGI Indy workstation [S1, unofficial].

## Sources
[S1] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
