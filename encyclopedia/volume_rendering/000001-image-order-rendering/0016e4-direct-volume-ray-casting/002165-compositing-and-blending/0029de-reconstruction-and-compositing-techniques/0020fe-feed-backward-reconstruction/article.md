# Feed-Backward Reconstruction

Also called: Gather-based reconstruction, Ray-casting approach

## Overview
Feed-backward reconstruction is a spatial-domain resampling method that computes each output sample by gathering weighted contributions from neighboring input samples. It solves the problem of mapping discrete 3D volume data to a continuous ray parameterization or output grid by centering a reconstruction kernel at the output location and integrating the input signal. It is distinguished from its sibling [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md) by operating in the spatial domain using discrete convolution kernels rather than composing trivariate polynomials in the algebraic domain.

## How it works
**Inputs:**
- Discrete input signal $f(i, j, k)$ representing volume samples.
- Reconstruction kernel $h(\cdot)$ with finite support.
- Output grid positions $x, y, z$ where values are to be computed.

**Outputs:**
- Resampled output signal $g(x, y, z)$.

**Data Structures:**
- Input array: Indexed by integer coordinates $(i, j, k)$.
- Output array: Indexed by floating-point or integer coordinates $(x, y, z)$.
- Kernel function: Evaluates weight $w$ for a given displacement.

**Steps:**
1. Initialize the output sample $g(x, y, z) = 0$.
2. Identify all input samples $(i, j, k)$ that lie within the support of the kernel centered at $(x, y, z)$.
3. For each such input sample, compute the displacement vector $\Delta = (x-i, y-j, z-k)$.
4. Evaluate the kernel weight $w = h(\Delta)$.
5. Accumulate the contribution: $g(x, y, z) \leftarrow g(x, y, z) + f(i, j, k) \cdot w$.

**Formula:**
The operation is defined by the discrete convolution sum [S1, unofficial]:
$$g(x, y, z) = \sum_{i=-\infty}^{\infty} \sum_{j=-\infty}^{\infty} \sum_{k=-\infty}^{\infty} f(i, j, k) h(x - i, y - j, z - k)$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def feed_backward_reconstruction(input_samples, kernel, output_positions):
    """
    input_samples: dict or array mapping (i,j,k) -> value
    kernel: function h(dx, dy, dz) -> weight
    output_positions: list of (x,y,z) tuples
    """
    output_values = {}
    
    for pos in output_positions:
        x, y, z = pos
        result = 0.0
        
        # Determine bounds of input samples under the kernel
        # [G] In practice, this requires knowing the kernel's support radius R.
        # We iterate over the bounding box of the kernel centered at pos.
        # The exact range depends on the specific kernel h.
        min_i = floor(x - R)
        max_i = ceil(x + R)
        min_j = floor(y - R)
        max_j = ceil(y + R)
        min_k = floor(z - R)
        max_k = ceil(z + R)
        
        for i in range(min_i, max_i + 1):
            for j in range(min_j, max_j + 1):
                for k in range(min_k, max_k + 1):
                    # Check if input sample exists at this location
                    if (i, j, k) in input_samples:
                        dx = x - i
                        dy = y - j
                        dz = z - k
                        weight = kernel(dx, dy, dz)
                        result += input_samples[(i, j, k)] * weight
        
        output_values[pos] = result
        
    return output_values
```

**Parameters:**
- Kernel support radius $R$: Determines the number of input samples accessed per output.
- Kernel type: e.g., Triangular pulse (linear interpolation) [S1, unofficial].

**Edge Cases:**
- **Kernel Extent:** If the kernel has infinite support, the summation must be truncated, introducing approximation errors.
- **Boundary Conditions:** Input samples outside the defined volume must be handled (e.g., clamping, zero-padding, or wrapping).
- **Precision:** Accumulating many small weights can lead to floating-point precision loss; Kahan summation may be used [G].

## Subtopics

### Spatial Convolution
The operation of convolving discrete samples with a reconstruction kernel in the spatial domain. It is preferred over frequency domain multiplication for discrete kernels of limited extent because transforming back and forth is more compute-intensive than direct spatial convolution [S1, unofficial]. The mathematical definition for 1D, 2D, and 3D signals is:
$$g(t) = \int_{-\infty}^{\infty} f(u)h(t-u) du$$
$$g(x, y) = \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} f(u, v) h(x - u, y - v) du dv$$
$$g(x, y, z) = \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} f(u, v, w) h(x - u, y - v, z - w) du dv dw$$
For discrete signals, this becomes the convolution sum [S1, unofficial].

### Feed-Forward Reconstruction
A reconstruction method that views convolution as distributing contributions from a given input sample to its neighboring output samples. It centers the reconstruction kernel at an input value and spreads the energy to each output lying under the kernel. It accesses each input sample once and each output sample many times, whereas feed-backward accesses each output once and each input many times [S1, unofficial]. Both methods produce identical results [S1, unofficial].

## Operating characteristics
- **Access Pattern:** Feed-backward reconstruction accesses each output sample once and each input sample many times [S1, unofficial].
- **Complexity:** The total number of accesses depends on the kernel extent and the number of intermediate samples. For example, with 4 input samples, 13 output samples, and linear reconstruction, feed-backward results in 39 sample accesses (2 input accesses per output $\times$ 13 outputs + 13 output writes) [S1, unofficial].
- **Performance:** Suitable for scenarios where output positions are fixed and known in advance, allowing efficient caching of input samples if the kernel support is small [G].
- **Limitations:** High memory bandwidth usage if the kernel support is large, as the same input samples may be read repeatedly for adjacent output samples.

## Background
Originated in the context of volume rendering and image reconstruction, feed-backward reconstruction is a standard technique for resampling discrete data. It is discussed in the context of parallel volume rendering algorithms [S1, unofficial]. The method is attributed to Feibush [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
