# Tensor-Specific Quantization

Also called: Fixed bit length encoding, Logarithmic quantization for core tensor

## Overview
Tensor-Specific Quantization is a fixed bit-length encoding strategy used to compactly represent Tucker decomposition coefficients, thereby reducing disk-to-host-to-device bandwidth during volume rendering [S1]. It distinguishes itself from generic floating-point storage by applying uniform linear quantization to factor matrices and logarithmic quantization to the core tensor, a design choice that simplifies parallel decoding on the GPU [S1].

## How it works
**Inputs and Outputs**
*   **Input:** The three factor matrices $U^{(1)}, U^{(2)}, U^{(3)}$ and the core tensor $B$ from a Tucker decomposition [S1].
*   **Output:** Quantized integer representations of these coefficients and the metadata required for reconstruction [S1].

**Data Structures**
The storage layout consists of:
1.  **Quantization Ranges:** Three floating-point numbers storing the global minimum and maximum for the factor matrices, and the maximum absolute value for the core tensor [S1].
2.  **Hot-Corner Value:** One floating-point number storing the unquantized value of $B(0,0,0)$ [S1].
3.  **Quantized Coefficients:**
    *   Factor matrices: Stored as $Q_U$-bit integers [S1].
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
import math

def quantize_tucker(factor_matrices, core_tensor, Q_U=8, Q_B=8, store_hot_corner=True):
    """
    Quantizes Tucker decomposition coefficients using tensor-specific quantization.
    """
    # 1. Determine ranges for factor matrices
    flat_factors = [x for U in factor_matrices for row in U for x in row]
    x_min = min(flat_factors)
    x_max = max(flat_factors)

    # 2. Quantize factor matrices
    quantized_factors = []
    denom_factors = x_max - x_min
    scale_factors = (2**Q_U - 1) / denom_factors if denom_factors != 0 else 0.0
    for U in factor_matrices:
        q_U = []
        for row in U:
            q_row = [int(round((x - x_min) * scale_factors)) for x in row]
            q_U.append(q_row)
        quantized_factors.append(q_U)

    # 3. Determine range for core tensor
    flat_core = [x for row in core_tensor for x in row]
    if store_hot_corner:
        hot_corner_val = flat_core[0]
        core_data_no_hot = flat_core[1:]
    else:
        hot_corner_val = None
        core_data_no_hot = flat_core

    x_max_core = max(abs(x) for x in core_data_no_hot) if core_data_no_hot else 0.0

    # 4. Quantize core tensor magnitude
    denom_core = math.log2(1 + x_max_core)
    scale_core = (2**Q_B - 1) / denom_core if denom_core != 0 else 0.0
    quantized_mag = []
    core_signs = []
    for i, row in enumerate(core_tensor):
        q_row = []
        s_row = []
        for j, x in enumerate(row):
            if store_hot_corner and i == 0 and j == 0:
                q_row.append(0)
                s_row.append(0)
            else:
                abs_x = abs(x)
                mag = math.log2(1 + abs_x) * scale_core
                q_row.append(int(round(mag)))
                s_row.append(1 if x >= 0 else 0)
        quantized_mag.append(q_row)
        core_signs.append(s_row)

    # 5. Metadata
    metadata = {
        'x_min': x_min,
        'x_max': x_max,
        'x_max_core': x_max_core,
        'hot_corner': hot_corner_val
    }

    return quantized_factors, quantized_mag, core_signs, metadata
```

**Edge Cases and Pitfalls**
*   **Hot-Corner:** The coefficient $B(0,0,0)$ often has a high absolute value close to the volume's norm [S1]. If included in the range calculation for logarithmic quantization, it compresses the dynamic range for the remaining coefficients [S1]. It is recommended to store this value separately as a float to maximize the quantization precision for the other coefficients [S1].
*   **Zero Range:** If $x_{max} = x_{min}$ for factor matrices or $|x_{max}| = 0$ for the core tensor, division by zero occurs [G]. These cases should be handled by setting the quantized values to 0 or a default constant [G].

## Operating characteristics
*   **Time Complexity:** Quantization is $O(N)$ where $N$ is the total number of coefficients [S1]. Decoding is also $O(N)$ and is simplified on the GPU due to fixed bit lengths [S1].
*   **Memory Complexity:** Storage is reduced from $N \cdot Q_{float}$ to $N \cdot Q_{bit} + \text{metadata}$ [S1]. Specifically, for a rank-$(R_1, R_2, R_3)$ approximation, storage is $R_1 R_2 R_3 Q_B + (I_1 R_1 + I_2 R_2 + I_3 R_3) Q_U$ bits, plus 3 floats for factor ranges and 1 float for the hot-corner [S1].
*   **Strengths:** Reduces bandwidth significantly; simplifies GPU parallel decoding; byte-aligned (8/16-bit) processing is efficient [S1].
*   **Limitations:** Quantization error is introduced, particularly for the core tensor where logarithmic mapping may not perfectly preserve relative differences for small values [S1].

## Background
Tensor-Specific Quantization was introduced in the context of Interactive Multiscale Tensor Reconstruction for Multiresolution Volume Rendering [S1]. It addresses the storage and bandwidth costs associated with maintaining Tucker decomposition coefficients as floating-point values [S1].

## Sources
[S1] Interactive Multiscale Tensor Reconstruction for Multiresolution Volume ..., https://ics.uci.edu/~gopi/SamplePubs/MultiscaleTensorVis.pdf, fetched 2026-10-26
