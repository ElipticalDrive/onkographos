# Uniform Quantization

Also called: Scalar uniform quantization, fixed-step quantization

## Overview
Uniform quantization is a lossy compression method that maps a continuous or high-precision discrete range of source data values to a limited set of discrete levels using constant spacing between levels [S1, S2, unofficial]. It solves the problem of reducing the memory footprint and bandwidth of volume data, typically achieving a 4:1 compression ratio by representing 32-bit floating-point data with 8-bit integers [S1, S2, unofficial]. It differs from [Non-uniform Quantization](5858-non-uniform-quantization.md) by allocating quantization levels with equal density across the entire data range, rather than adapting level density to the source data distribution or user-defined importance [S1, S2, unofficial].

## How it works
**Inputs and outputs:**
- Input: A volume dataset $D$ where each voxel value $x$ is a real number (typically 32-bit float).
- Output: A compressed dataset $D'$ where each voxel value is an integer index $q \in \{0, 1, \dots, M-1\}$, where $M$ is the number of quantization levels.

**Data structures:**
- The compressed data is stored as a linear array of integers. The bit-width of the integer is determined by $M$. For a standard 4:1 compression, $M=256$ and the storage is 8-bit unsigned integers (`uint8`).
- Metadata required for decoding includes:
  - $x_{min}$: The minimum value of the source data range.
  - $x_{max}$: The maximum value of the source data range.
  - $M$: The number of quantization levels (e.g., 256).

**Parameters:**
- $M$: Number of quantization levels. Commonly 256 for 8-bit representation [S1, S2, unofficial].
- Step size $\phi$: The width of each quantization bin, calculated as $\phi = \frac{x_{max} - x_{min}}{M}$ [S1, S2, unofficial].

**Algorithm:**
1. Determine the global minimum $x_{min}$ and maximum $x_{max}$ of the input volume data $D$.
2. Calculate the step size $\phi = \frac{x_{max} - x_{min}}{M}$.
3. For each voxel value $x$ in $D$:
   a. Compute the quantized index $q$ using the formula:
      $$ q = \left\lfloor \frac{x - x_{min}}{\phi} \right\rfloor $$
   b. Clamp $q$ to the range $[0, M-1]$ to handle values outside $[x_{min}, x_{max}]$ [G].
   c. Store $q$ in the output array.

**Decoding:**
To reconstruct an approximation $\hat{x}$ from a quantized index $q$:
$$ \hat{x} = x_{min} + q \cdot \phi + \frac{\phi}{2} $$
This represents the midpoint of the quantization bin [G].

**Error Measure:**
For uniformly distributed source data values $x$, the mean squared error (MSE) of uniform quantization is given by [S2, unofficial]:
$$ \sigma^2 = \int_{x_{min}}^{x_{max}} \frac{1}{M} (x - \phi_i)^2 dx $$
where $\phi_i$ is the representative value for the $i$-th level (typically the bin center) and $f(x) = \frac{1}{x_{max}-x_{min}}$ is the probability density function for uniform distribution [S2, unofficial].

For visualization tasks, the error measure incorporates an importance function $\alpha(x)$, such as the opacity transfer function provided by the user, modifying the equation to [S1, S2, unofficial]:
$$ \sigma^2 = E \int (x - \phi(x))^2 f(x) \alpha(x) dx $$
where $f(x)$ characterizes the general source data distribution and $\alpha(x)$ is the importance function [S1, S2, unofficial].

**Pitfalls:**
- Wide dynamic range in data (e.g., turbulence flow) can cause important features in later time steps to become invisible if uniform quantization is used without care [S1, S2, unofficial].
- If the data distribution is not uniform, uniform quantization may waste levels on empty ranges and under-resolve regions of interest [S1, S2, unofficial].

## Subtopics

### [Non-uniform Quantization](5858-non-uniform-quantization.md)
Allocates more quantization levels to specific ranges to enhance features and minimize distortion error for data with wide dynamic ranges. Choose this when data distribution is skewed or when specific features must be preserved at the expense of others.

## Operating characteristics
- **Time Complexity:** $O(N)$ for encoding and decoding, where $N$ is the number of voxels. Requires a single pass to find min/max and another pass to quantize.
- **Memory Complexity:** $O(N)$ for storing the compressed data. Reduces memory usage by a factor of 4 compared to 32-bit floats (for 8-bit quantization) [S1, S2, unofficial].
- **Performance:** Simple to implement and fast to execute. No complex lookup tables or adaptive statistics calculation required during encoding [S1, S2, unofficial].
- **Strengths:** Low computational overhead, simple hardware implementation, predictable compression ratio [S1, S2, unofficial].
- **Limitations:** Poor performance on data with wide dynamic ranges or non-uniform distributions, where important features may be lost or aliased [S1, S2, unofficial].
- **When to use:** Suitable for data with relatively uniform distribution and narrow dynamic range, or when computational resources are limited [S1, S2, unofficial].
- **When not to use:** Unsuitable for data with wide dynamic ranges (e.g., turbulence flow) where non-uniform or adaptive quantization is required to preserve features [S1, S2, unofficial].

## Background
Uniform quantization is a well-studied area in signal processing and data compression [S1, S2, unofficial]. In the context of volume rendering, its impact was carefully studied in the late 1990s, particularly for time-varying volume data generated from numerical simulations [S1, S2, unofficial]. The method is often used as a baseline for comparison with more advanced techniques like non-uniform and adaptive quantization [S1, S2, unofficial]. It is frequently followed by spatial encoding methods such as octree encoding to further compress the data [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
