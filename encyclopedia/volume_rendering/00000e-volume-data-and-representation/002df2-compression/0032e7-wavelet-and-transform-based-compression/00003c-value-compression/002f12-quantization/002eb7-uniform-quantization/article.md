# Uniform Quantization

Also called: Scalar uniform quantization, fixed-step quantization

## Overview
Uniform quantization is a lossy compression method that maps a continuous or high-precision input domain to a discrete set of $M$ levels with equal spacing [S1, S2, unofficial]. It solves the problem of reducing the bit-depth of volume data (e.g., from 32-bit floats to 8-bit integers) to facilitate storage and transmission [S1, S2, unofficial]. It differs from non-uniform quantization by using constant step sizes across the entire data range, making it simple to implement but potentially ineffective for data with wide dynamic ranges where important features may be obscured [S1, S2, unofficial].

## How it works
Uniform quantization operates on the scalar values of the volume data after they have been decomposed into coefficients by the parent transform [Parent](5935-quantization.md).

**Inputs and Outputs:**
*   **Input:** A set of source data values $x$ (typically transform coefficients) with a known minimum $x_{min}$ and maximum $x_{max}$.
*   **Output:** A set of quantized integer indices $q \in \{0, 1, \dots, M-1\}$.

**Data Structures:**
The output is typically stored as a compact array of integers. If the input is 32-bit floating point and the target is 8-bit, the output array uses 8 bits per value, achieving a 4:1 compression ratio [S1, S2, unofficial].

**Parameters:**
*   $M$: The number of quantization levels. Typically $M = 2^b$ where $b$ is the target bit-depth (e.g., $M=256$ for 8-bit) [S1, S2, unofficial].
*   $x_{min}$: The minimum value in the data range.
*   $x_{max}$: The maximum value in the data range.
*   $\phi$: The step size (quantization interval), defined as $\phi = \frac{x_{max} - x_{min}}{M}$ [S1, S2, unofficial].

**Formulas:**
The quantization function $\phi(x)$ maps a value $x$ to a reconstruction level. For uniform quantization of uniformly distributed source data, the mean squared error (MSE) is given by [S1, S2, unofficial]:

$$
\sigma^2 = \int_{x_{min}}^{x_{max}} \frac{1}{M} (x - \phi_i)^2 dx
$$

where $f(x) = \frac{1}{x_{max} - x_{min}}$ is the probability density function for uniformly distributed data [S2, unofficial].

For visualization tasks, the error measure is modified to include an importance function $\alpha(x)$, such as the opacity transfer function provided by the user [S1, S2, unofficial]:

$$
\sigma^2 = E \int (x - \phi(x))^2 f(x) \alpha(x) dx
$$

where $f(x)$ characterizes the general source data distribution and $\alpha(x)$ is the importance function [S1, S2, unofficial].

**Algorithm:**
1.  Determine the global or local range $[x_{min}, x_{max}]$ of the input data.
2.  Calculate the step size $\phi = \frac{x_{max} - x_{min}}{M}$.
3.  For each input value $x$:
    a. Compute the index $q = \lfloor \frac{x - x_{min}}{\phi} \rfloor$.
    b. Clamp $q$ to the range $[0, M-1]$ to handle values outside the defined range.
    c. Store $q$ as the quantized value.
4.  [G] To reconstruct, the value is typically recovered as $x' = x_{min} + (q + 0.5)\phi$ to minimize average error, though the specific reconstruction formula is not explicitly detailed in the sources [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniform_quantize(data, num_levels):
    """
    Quantizes a 1D array of data values using uniform quantization.
    
    Args:
        data: List or array of float values.
        num_levels: Integer M, the number of quantization levels (e.g., 256).
        
    Returns:
        List of integers representing the quantized indices.
    """
    if not data:
        return []
        
    x_min = min(data)
    x_max = max(data)
    
    # Avoid division by zero if all values are identical
    if x_min == x_max:
        return [0] * len(data)
        
    step_size = (x_max - x_min) / num_levels
    quantized_values = []
    
    for x in data:
        # Calculate index
        index = int((x - x_min) / step_size)
        
        # Clamp to valid range [0, num_levels - 1]
        if index < 0:
            index = 0
        elif index >= num_levels:
            index = num_levels - 1
            
        quantized_values.append(index)
        
    return quantized_values
```

**Edge Cases and Pitfalls:**
*   **Wide Dynamic Range:** If the data has a wide dynamic range (e.g., turbulence flow data), uniform quantization can cause important features in later time steps to become invisible because the quantization steps are too large for the smaller values [S1, S2, unofficial].
*   **Clamping:** Values outside $[x_{min}, x_{max}]$ must be clamped, which introduces distortion at the boundaries [G].

## Subtopics

### [Non-uniform Quantization](5858-non-uniform-quantization.md)
Allocates more quantization levels to specific ranges of source data values to enhance features and minimize distortion, preferred for data with wide dynamic ranges where uniform spacing fails.

## Operating characteristics
*   **Complexity:** Time complexity is $O(N)$ where $N$ is the number of data points, as it requires a single pass to determine range and a single pass to quantize. Memory complexity is $O(N)$ for the output array, reduced by the bit-depth ratio (e.g., 4:1 for 32-bit to 8-bit) [S1, S2, unofficial].
*   **Performance:** Achieves a 4:1 compression ratio by representing 32-bit data with 8 bits [S1, S2, unofficial].
*   **Strengths:** Simple to implement; computationally inexpensive; effective for data with narrow dynamic ranges or uniform distributions [S1, S2, unofficial].
*   **Limitations:** Ineffective for data with wide dynamic ranges or non-uniform distributions, as it cannot allocate more precision to regions of interest [S1, S2, unofficial].
*   **When to use:** When data distribution is uniform or dynamic range is small, and computational simplicity is prioritized [S1, S2, unofficial].
*   **When not to use:** When data has a wide dynamic range (e.g., turbulence flow) or when preserving specific features is critical, as non-uniform or adaptive quantization is required [S1, S2, unofficial].

## Background
Uniform quantization is described as the "simplest lossy compression method" in the context of volume rendering [S1, S2, unofficial]. It is a well-studied area in signal processing, but its specific impact on volume rendering was noted as not being carefully studied in the cited literature [S1, S2, unofficial]. The method is often used as a default in renderers, though non-uniform and adaptive quantizers are shown to be more effective for feature detection [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
