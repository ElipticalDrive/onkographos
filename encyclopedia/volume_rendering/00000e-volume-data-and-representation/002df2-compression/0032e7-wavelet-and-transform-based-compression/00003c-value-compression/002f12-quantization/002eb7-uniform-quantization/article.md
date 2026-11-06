# Uniform Quantization


## Overview
Uniform quantization is a lossy compression method that maps continuous or high-bit-depth volume data to a finite set of discrete levels using equal spacing, typically achieving a 4:1 compression ratio by representing 32-bit data with 8 bits [S1, unofficial][S2, unofficial]. It differs from [Non-uniform Quantization](5858-non-uniform-quantization.md) by distributing quantization levels evenly across the data domain rather than adapting to data distribution or importance, making it simple but potentially ineffective for data with wide dynamic ranges [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volume dataset with values $x$ in the range $[x_{min}, x_{max}]$, typically 32-bit floating point.
- Output: A compressed dataset where each value is represented by an integer index $i$ in the range $[0, M-1]$, where $M$ is the number of quantization levels.

**Data Structures:**
- The compressed data is stored as a sequence of integers. If 8-bit quantization is used, each voxel is stored as a single byte.
- The quantization step size $\phi$ is a scalar parameter derived from the input range and $M$.

**Parameters:**
- $M$: Number of quantization levels. Commonly $M=256$ for 8-bit representation [S1, unofficial][S2, unofficial].
- $x_{min}$: Minimum value in the dataset.
- $x_{max}$: Maximum value in the dataset.
- $\phi$: Quantization step size, calculated as $\phi = (x_{max} - x_{min}) / M$ [S2, unofficial].

**Steps:**
1. Determine the global minimum $x_{min}$ and maximum $x_{max}$ of the input volume data.
2. Choose the number of quantization levels $M$ (e.g., 256 for 8-bit) [S1, unofficial][S2, unofficial].
3. Calculate the step size $\phi = (x_{max} - x_{min}) / M$ [S2, unofficial].
4. For each voxel value $x$:
   a. Compute the quantized index $i = \lfloor (x - x_{min}) / \phi \rfloor$.
   b. Clamp $i$ to the range $[0, M-1]$ to handle values outside $[x_{min}, x_{max}]$ [G].
   c. Store $i$ as the compressed value.
5. To reconstruct, compute $x' = x_{min} + (i + 0.5) \phi$ [G]. This midpoint reconstruction minimizes mean squared error for uniform distributions.

**Formulas:**
The mean squared error (MSE) for uniform quantization of uniformly distributed source data values $x$ is given by [S2, unofficial]:
$$ \sigma^2 = \int_{x_{min}}^{x_{max}} \frac{1}{M} (x - \phi_i)^2 dx $$
where $f(x) = 1/(x_{max}-x_{min})$ is the probability density function for uniformly distributed source data [S2, unofficial].

For visualization tasks, the error measure includes an importance function $\alpha(x)$, typically the opacity transfer function provided by the user, modifying the equation to [S1, unofficial][S2, unofficial]:
$$ \sigma^2 = E \int (x - \phi(x))^2 f(x) \alpha(x) dx $$
where $f(x)$ characterizes a general source data distribution and $\alpha(x)$ is the importance function [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls:**
- Wide dynamic range data (e.g., turbulence flow) can cause important features in later time steps to become invisible if uniform quantization is used without care, as the quantization levels may not allocate sufficient resolution to ranges of interest [S1, unofficial][S2, unofficial].
- Values outside the range $[x_{min}, x_{max}]$ must be clamped to avoid out-of-bounds indices [G].

## Subtopics

### [Non-uniform Quantization](5858-non-uniform-quantization.md)
Allocates quantization levels non-uniformly to minimize perceptual distortion or enhance specific features, preferred for data with wide dynamic ranges where uniform quantization fails to preserve important details.

## Operating characteristics
- **Time Complexity:** $O(N)$ for quantization and reconstruction, where $N$ is the number of voxels.
- **Memory Complexity:** $O(N \cdot b)$, where $b$ is the number of bits per voxel after quantization (e.g., 8 bits).
- **Compression Ratio:** Typically 4:1 when reducing 32-bit data to 8-bit [S1, unofficial][S2, unofficial].
- **Strengths:** Simple implementation, low computational overhead, predictable error distribution for uniformly distributed data.
- **Limitations:** Ineffective for data with wide dynamic ranges or non-uniform distributions, as it does not adapt to data statistics or user-defined importance functions [S1, unofficial][S2, unofficial].
- **When to Use:** Suitable for data with narrow dynamic ranges or uniform distributions where simplicity and speed are prioritized over feature preservation [S1, unofficial][S2, unofficial].
- **When Not to Use:** Unsuitable for data with wide dynamic ranges (e.g., turbulence flow) where important features may be lost [S1, unofficial][S2, unofficial].

## Background
Uniform quantization is a well-studied area in lossy compression, applied to volume rendering to reduce data size for efficient encoding and rendering [S1, unofficial][S2, unofficial]. It was highlighted in the context of time-varying volume data, where it serves as a baseline for comparison with non-uniform and adaptive quantizers [S1, unofficial][S2, unofficial]. The method is often used as a default in renderers, though non-uniform and adaptive quantization are preferred for minimizing distortion error and enhancing feature detection in complex datasets [S1, unofficial][S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
