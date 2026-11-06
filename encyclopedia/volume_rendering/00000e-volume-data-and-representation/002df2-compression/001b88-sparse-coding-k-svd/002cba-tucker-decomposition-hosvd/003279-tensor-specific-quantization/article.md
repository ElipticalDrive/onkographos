# Tensor-Specific Quantization

Also called: Fixed bit length encoding, Logarithmic quantization for core tensor

## Overview
Tensor-Specific Quantization is a fixed bit-length encoding strategy used to compactly represent the coefficients of a Tucker decomposition, thereby reducing disk-to-host-to-device bandwidth during volume rendering [S1]. It distinguishes itself from generic floating-point storage by applying uniform linear quantization to factor matrices and logarithmic quantization to the core tensor, a design choice that simplifies parallel decoding on the GPU [S1].

## How it works
**Inputs and Outputs**
*   **Input:** The Tucker decomposition components of a volume dataset: three factor matrices $U^{(1)}, U^{(2)}, U^{(3)}$ and an N-dimensional core tensor $B$.
*   **Output:** Quantized integer representations of these coefficients, along with metadata required for reconstruction.

**Data Structures**
The storage layout consists of:
1.  **Quantization Ranges:** Three floating-point numbers storing the global minimum and maximum for the factor matrices, and the maximum absolute value for the core tensor.
2.  **Hot-Corner Value:** One floating-point number storing the unquantized value of $B(0,0,0)$ [S1].
3.  **Quantized Coefficients:**
    *   Factor matrices: Stored as $Q_U$-bit integers.
    *   Core tensor: Stored as $Q_B$-bit integers for magnitude, plus a separate sign bit for each coefficient [S1].

**Parameters**
*   $Q_U$: Number of bits for factor matrix quantization. Typical values are 8 or 16 bits [S1].
*   $Q_B$: Number of bits for core tensor magnitude quantization. Typical values are 8 or 16 bits [S1].
*   Range bounds: $x_{min}$ and $x_{max}$ define the range for factor matrices. $|x_{max}|$ defines the range for the core tensor [S1].

**Steps**
1.  **Determine Ranges:** Calculate $x_{min}$ and $x_{max}$ from all coefficients in the three factor matrices. Calculate $|x_{max}|$ as the maximum absolute value among all core tensor coefficients, excluding the hot-corner if stored separately [S1].
2.  **Quantize Factor Matrices:** For each coefficient $x$ in $U^{(n)}$, apply linear quantization:
    $$ \tilde{x}_U = (2^{Q_U} - 1) \cdot \frac{x - x_{min}}{x_{max} - x_{min}} $$
    Store $\tilde{x}_U$ as an integer [S1].
3.  **Quantize Core Tensor Magnitude:** For each coefficient $x$ in $B$ (excluding the hot-corner if handled separately), calculate the magnitude quantization:
    $$ |\tilde{x}_B| = (2^{Q_B} - 1) \cdot \frac{\log_2(1 + |x|)}{\log_2(1 + |x_{max}|)} $$
    Store $|\tilde{x}_B|$ as an integer [S1].
4.  **Store Sign:** Store the sign of each core tensor coefficient $x$ in a separate bit [S1].
5.  **Store Hot-Corner:** If the hot-corner optimization is enabled, store the original floating-point value of $B(0,0,0)$ separately [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quantize_tucker(factor_matrices, core_tensor, Q_U=8, Q_B=8, store_hot_corner=True):
    """
    Quantizes Tucker decomposition coefficients using tensor-specific quantization.
    
    Args:
        factor_matrices: List of 3 matrices U^(1), U^(2), U^(3).
        core_tensor: 3D tensor B.
        Q_U: Bits for factor matrices.
        Q_B: Bits for core tensor magnitude.
        store_hot_corner: Whether to store B(0,0,0) as float.
        
    Returns:
        quantized_factors: List of quantized factor matrices.
        quantized_core_mag: Quantized magnitude tensor.
        core_signs: Sign bits for core tensor.
        metadata: Dict with ranges and hot-corner value.
    """
    # 1. Determine ranges for factor matrices
    all_factors = [f.flatten() for f in factor_matrices]
    flat_factors = np.concatenate(all_factors)
    x_min = np.min(flat_factors)
    x_max = np.max(flat_factors)
    
    # 2. Quantize factor matrices
    quantized_factors = []
    for U in factor_matrices:
        # Linear quantization: map [x_min, x_max] to [0, 2^Q_U - 1]
        # Note: Source uses (2^Q_U - 1) scaling factor
        scale = (2**Q_U - 1) / (x_max - x_min)
        q_U = np.round((U - x_min) * scale).astype(np.uint16) # Use uint16 to hold 16-bit values
        quantized_factors.append(q_U)
        
    # 3. Determine range for core tensor
    # Exclude hot-corner from max calculation if storing it separately to preserve range for others
    core_data = core_tensor.flatten()
    if store_hot_corner:
        # Hot corner is at index 0 in flattened tensor if row-major and B(0,0,0) is first
        # Assuming standard tensor indexing B(0,0,0) corresponds to first element
        hot_corner_val = core_data[0]
        core_data_no_hot = core_data[1:]
    else:
        hot_corner_val = None
        core_data_no_hot = core_data
        
    x_max_core = np.max(np.abs(core_data_no_hot))
    
    # 4. Quantize core tensor magnitude
    # Logarithmic quantization
    # Avoid division by zero if x_max_core is 0
    if x_max_core == 0:
        scale_core = 0
    else:
        scale_core = (2**Q_B - 1) / np.log2(1 + x_max_core)
        
    # Calculate magnitude for all core coefficients
    abs_core = np.abs(core_tensor)
    # Apply log scaling
    # log2(1 + |x|) / log2(1 + |x_max|) * (2^Q_B - 1)
    # Handle the case where x_max_core is 0 separately to avoid NaN/Inf
    if x_max_core > 0:
        log_vals = np.log2(1 + abs_core)
        log_max = np.log2(1 + x_max_core)
        quantized_mag = np.round(log_vals / log_max * (2**Q_B - 1)).astype(np.uint16)
    else:
        quantized_mag = np.zeros_like(core_tensor, dtype=np.uint16)
        
    # 5. Store signs
    core_signs = np.sign(core_tensor) # 1 for positive, -1 for negative, 0 for zero
        
    # 6. Metadata
    metadata = {
        'x_min': x_min,
        'x_max': x_max,
        'x_max_core': x_max_core,
        'hot_corner': hot_corner_val
    }
    
    return quantized_factors, quantized_mag, core_signs, metadata
```

**Edge Cases and Pitfalls**
*   **Hot-Corner:** The coefficient $B(0,0,0)$ often has a high absolute value close to the volume's norm. If included in the range calculation for logarithmic quantization, it compresses the dynamic range for the remaining coefficients. It is recommended to store this value separately as a float to maximize the quantization precision for the other coefficients [S1].
*   **Zero Range:** If $x_{max} = x_{min}$ for factor matrices or $|x_{max}| = 0$ for the core tensor, division by zero occurs. These cases should be handled by setting the quantized values to 0 or a default constant [G].

## Operating characteristics
*   **Time Complexity:** Quantization is $O(N)$ where $N$ is the total number of coefficients. Decoding is also $O(N)$ and is simplified on the GPU due to fixed bit lengths [S1].
*   **Memory Complexity:** Storage is reduced from $N \cdot Q_{float}$ to $N \cdot Q_{bit} + \text{metadata}$. Specifically, for a rank-$(R_1, R_2, R_3)$ approximation, storage is $R_1 R_2 R_3 Q_B + (I_1 R_1 + I_2 R_2 + I_3 R_3) Q_U$ bits, plus 3 floats for factor ranges and 1 float for the hot-corner [S1].
*   **Strengths:** Reduces bandwidth significantly; simplifies GPU parallel decoding; byte-aligned (8/16-bit) processing is efficient [S1].
*   **Limitations:** Quantization error is introduced, particularly for the core tensor where logarithmic mapping may not perfectly preserve relative differences for small values [S1].

## Background
Tensor-Specific Quantization was introduced in the context of Interactive Multiscale Tensor Reconstruction for Multiresolution Volume Rendering [S1]. It addresses the storage and bandwidth costs associated with maintaining Tucker decomposition coefficients as floating-point values [S1].

## Sources
[S1] Interactive Multiscale Tensor Reconstruction for Multiresolution Volume ..., https://ics.uci.edu/~gopi/SamplePubs/MultiscaleTensorVis.pdf, fetched 2026-10-26
