# Uniform Quantization


## Overview
Uniform quantization is a lossy compression method that maps a continuous or high-bit-depth source domain to a discrete set of indices using evenly spaced intervals, serving as the baseline for volume data compression [S1, S2, unofficial]. It solves the problem of reducing memory footprint and bandwidth by representing large raw data values (e.g., 32-bit floats) with a limited number of bits (e.g., 8 bits), typically achieving a 4:1 compression ratio [S1, S2, unofficial]. Unlike non-uniform or adaptive quantization, it does not allocate levels based on data distribution or user-defined importance, making it simple to implement but potentially ineffective for data with wide dynamic ranges where important features may become invisible [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A source data value $x$ (typically a scalar field value such as density or temperature) and the global or local range bounds $[x_{min}, x_{max}]$.
*   **Output:** A discrete index $i$ (integer) representing the quantized value, and optionally the reconstructed value $R(i)$.

**Data Structures:**
*   **Source Data:** Array of floating-point values (e.g., 32-bit IEEE 754).
*   **Quantized Data:** Array of integers (e.g., 8-bit unsigned integers) with $M = 2^b$ levels, where $b$ is the target bit depth.
*   **Metadata:** Two scalar values $x_{min}$ and $x_{max}$ defining the range. If local quantization is used, these are stored per-block.

**Parameters:**
*   $M$: Number of quantization levels. Typically $M=256$ for 8-bit quantization of 32-bit data [S1, S2, unofficial].
*   $b$: Target bit depth (e.g., 8) [S1, S2, unofficial].
*   $x_{min}, x_{max}$: Minimum and maximum values of the source data range [S1, S2, unofficial].

**Formulas:**
The step size $\phi$ is defined as:
$$ \phi = \frac{x_{max} - x_{min}}{M} $$

The quantization function $Q(x)$ maps $x$ to an index $i$:
$$ i = Q(x) = \left\lfloor \frac{x - x_{min}}{\phi} \right\rfloor $$
Clamping is required to ensure $0 \le i < M$ [G].

The reconstruction function $R(i)$ maps the index back to an approximate value. The sources define the error metric using the center of the bin, implying the reconstruction value is the midpoint of the interval $[x_{min} + i\phi, x_{min} + (i+1)\phi]$:
$$ R(i) = x_{min} + i \cdot \phi + \frac{\phi}{2} $$

**Numbered Steps:**
1.  Determine the source range $[x_{min}, x_{max}]$ for the data block or volume.
2.  Calculate the number of levels $M = 2^b$ based on the target bit depth $b$.
3.  Compute the step size $\phi = (x_{max} - x_{min}) / M$.
4.  For each source value $x$:
    a. Compute the raw index: $idx = (x - x_{min}) / \phi$.
    b. Clamp $idx$ to the integer range $[0, M-1]$.
    c. Store the integer index $i$ in the compressed data structure.
5.  Store $x_{min}$ and $x_{max}$ in the header/metadata.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniform_quantize(source_value, x_min, x_max, num_levels):
    """
    Quantizes a single source value using uniform quantization.
    
    Args:
        source_value (float): The original data value x.
        x_min (float): Minimum value of the range.
        x_max (float): Maximum value of the range.
        num_levels (int): Number of quantization levels M (e.g., 256).
        
    Returns:
        int: The quantized index i.
    """
    # Calculate step size
    phi = (x_max - x_min) / num_levels
    
    # Calculate raw index
    # Guard against division by zero if x_min == x_max
    if phi == 0:
        return 0
        
    raw_index = (source_value - x_min) / phi
    
    # Clamp to valid range [0, num_levels - 1]
    # This handles values outside [x_min, x_max]
    index = int(raw_index)
    if index < 0:
        index = 0
    elif index >= num_levels:
        index = num_levels - 1
        
    return index

def uniform_reconstruct(index, x_min, x_max, num_levels):
    """
    Reconstructs the approximate value from the quantized index.
    
    Args:
        index (int): The quantized index i.
        x_min (float): Minimum value of the range.
        x_max (float): Maximum value of the range.
        num_levels (int): Number of quantization levels M.
        
    Returns:
        float: The reconstructed value R(i).
    """
    phi = (x_max - x_min) / num_levels
    # Reconstruction is the center of the bin
    reconstructed_value = x_min + index * phi + phi / 2
    return reconstructed_value
```

**Edge Cases and Pitfalls:**
*   **Wide Dynamic Range:** If the data has a wide dynamic range (e.g., turbulence flow), uniform quantization may cause important features in later time steps or low-magnitude regions to become invisible because the step size $\phi$ is too large for those regions [S1, S2, unofficial].
*   **Out-of-Range Values:** Values outside $[x_{min}, x_{max}]$ must be clamped, which introduces distortion [G].
*   **Constant Blocks:** If $x_{min} = x_{max}$, $\phi = 0$, leading to division by zero. This should be handled by returning a constant index (e.g., 0) [G].

## Subtopics

### [Non-uniform Quantization](5858-non-uniform-quantization.md)
Allocates levels non-uniformly to minimize perceptual distortion or enhance specific features, preferred for data with wide dynamic ranges where uniform spacing fails.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per value for quantization and reconstruction.
*   **Memory Complexity:** Reduces storage from $B_{source}$ bits to $B_{target}$ bits per value. The overhead is $O(1)$ for storing $x_{min}$ and $x_{max}$ per block or volume.
*   **Compression Ratio:** Typically 4:1 when converting 32-bit data to 8-bit data [S1, S2, unofficial].
*   **Strengths:** Simple implementation, low computational overhead, predictable memory usage.
*   **Limitations:** Ineffective for data with wide dynamic ranges or non-uniform distributions where important features are lost due to coarse quantization in critical regions [S1, S2, unofficial].
*   **When to Use:** When data distribution is uniform or when computational simplicity is prioritized over fidelity for wide-range data.
*   **When Not to Use:** For data with wide dynamic ranges (e.g., turbulence) where non-uniform or adaptive quantization is required to preserve features [S1, S2, unofficial].

## Background
Uniform quantization is a well-studied area in signal processing and data compression [S1, S2, unofficial]. In the context of volume rendering, its impact was carefully studied in the late 1990s, specifically in the work by [S1, S2, unofficial] on efficient encoding and rendering of time-varying volume data. The sources highlight that while uniform quantization is the default in many renderers, it is often insufficient for complex scientific datasets, leading to the development of non-uniform and adaptive quantizers [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
