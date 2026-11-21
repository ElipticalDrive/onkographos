# Feed-Backward Reconstruction

Also called: Gather-based reconstruction, Ray-casting approach

## Overview
Feed-backward reconstruction is a volume rendering technique that views convolution as gathering contributions from neighboring input samples for each output sample [S1, unofficial]. It calculates each output value individually by centering the reconstruction kernel at the output sample and computing a weighted average of input samples lying under the kernel [S1, unofficial]. It is distinguished from its sibling [Feed-Forward Reconstruction](2513-spatial-convolution.md) by its access pattern: it accesses each output sample once and each input sample many times, whereas feed-forward methods access each input sample once and each output sample many times [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sequence of discrete input samples $X_j$ (where $j$ indexes the input domain, e.g., voxels or ray samples).
- Input: A reconstruction kernel $K(u)$, defined by its shape (e.g., triangular, B-spline) and support width.
- Input: A set of output sample positions $I$ (e.g., pixel centers or ray sample locations).
- Output: A sequence of output values $A_i$ corresponding to each position $I$.

**Data Structures:**
- `input_samples`: Array of values $X_j$.
- `kernel`: Function or lookup table defining $K(u)$.
- `output_positions`: Array of coordinates $I$.
- `output_samples`: Array of values $A_i$, initialized to zero.

**Steps:**
1. Initialize all output samples $A_i = 0$ for all $i$.
2. For each output sample position $I$:
    a. Initialize accumulator $A_i = 0$.
    b. Identify all input samples $X_j$ that lie within the support of the kernel centered at $I$.
    c. For each such input sample $X_j$:
        i. Calculate the relative distance $u = I - X_j$ (or appropriate coordinate transformation).
        ii. Evaluate the kernel weight $w = K(u)$.
        iii. Update the output sample: $A_i \leftarrow A_i + X_j \cdot w$.
3. Return the array of output samples $A_i$.

**Formulas:**
The value of output sample $i$ is given by:
$$
A_i = \sum_{j \in \text{support}(I)} X_j \cdot K(I - X_j)
$$
Where:
- $A_i$ is the value of the $i$-th output sample.
- $X_j$ is the value of the $j$-th input sample.
- $K(\cdot)$ is the reconstruction kernel function.
- $\text{support}(I)$ is the set of input indices $j$ such that $K(I - X_j) \neq 0$.

**Parameters:**
- Kernel type: Commonly linear (triangular pulse) for simplicity, but can be higher-order B-splines [S1, unofficial].
- Kernel support: Determines the number of input samples accessed per output sample. For linear reconstruction, the support is typically 2 input samples per output sample if sampling rates are comparable, but varies with resolution [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Kernel Support Boundaries:** Ensure that the loop over input samples correctly handles cases where the kernel support extends beyond the input data boundaries. Samples outside the input range should be treated as zero or handled via boundary conditions [G].
- **Precision:** Accumulating many small weights can lead to floating-point precision issues. Use compensated summation if high precision is required and the number of contributing samples is large [G].
- **Degenerate Input:** If input samples are not uniformly spaced, the distance calculation $I - X_j$ must account for the actual coordinate of $X_j$, not just the index [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def feed_backward_reconstruction(input_samples, kernel_func, output_positions):
    """
    Performs feed-backward reconstruction.
    
    Args:
        input_samples: List of input values X_j.
        kernel_func: Function K(u) that returns the kernel weight for distance u.
        output_positions: List of output positions I.
        
    Returns:
        List of output values A_i.
    """
    num_outputs = len(output_positions)
    output_samples = [0.0] * num_outputs
    
    for i, I in enumerate(output_positions):
        accumulator = 0.0
        # Identify input samples within kernel support
        # In practice, this is often optimized by iterating over a local window
        # of input samples around I.
        for j, X_j in enumerate(input_samples):
            # Calculate distance from output position to input sample
            u = I - X_j
            
            # Evaluate kernel
            weight = kernel_func(u)
            
            # If weight is non-zero, accumulate
            if weight != 0.0:
                accumulator += X_j * weight
        
        output_samples[i] = accumulator
        
    return output_samples
```

## Subtopics

### [Spatial Convolution](2513-spatial-convolution.md)
Spatial convolution is a neighborhood-based operation where output values are calculated as a weighted average of neighboring input samples using a filter kernel. In volume rendering, it is used to convolve discrete samples with a reconstruction kernel in the spatial domain. It is distinguished from frequency-domain methods (like FFT-based convolution) by operating directly on spatial data, which is preferred for discrete kernels of limited extent because transforming to and from the frequency domain is more compute-intensive than direct spatial convolution.

### [Area-based Pixel Kernel](14355-area-based-pixel-kernel.md)
The Area-based Pixel Kernel is a reconstruction kernel used in feed-backward reconstruction that assigns pixel contributions proportional to the geometric area of intersection between a pixel and a thick projection line. It solves the artifact issues inherent in Dirac and B-Spline kernels, which leave unreconstructed pixels during subsampling, and linear kernels, which fail to fully exploit available data during oversampling. It is distinguished from its sibling [Spatial Convolution](2513-spatial-convolution.md) by its specific geometric interpretation of projection thickness and its optimization via rotation-based approximation.

### Feed-Forward Reconstruction
Feed-forward reconstruction is a reconstruction method that views convolution as distributing contributions from a given input sample to its neighboring output samples [S1, unofficial]. It centers the reconstruction kernel at an input value and spreads the energy to each output lying under the kernel [S1, unofficial]. It is distinguished from feed-backward reconstruction by its access pattern: it accesses each input sample once and each output sample many times [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time Complexity: $O(N_{out} \cdot N_{kernel})$, where $N_{out}$ is the number of output samples and $N_{kernel}$ is the number of input samples within the kernel support for each output.
- Memory Complexity: $O(N_{in} + N_{out})$ for storing input and output samples.

**Measured Performance:**
- In an example with 4 input samples, 13 output samples, and linear reconstruction, feed-backward reconstruction results in 39 sample accesses (2 input accesses per output * 13 outputs + 13 output writes) [S1, unofficial].
- Feed-forward reconstruction in the same scenario results in 24 sample accesses (5 output accesses per input * 4 inputs + 4 input reads) [S1, unofficial].

**Scaling:**
- The total number of accesses depends on the kernel extent and the number of intermediate samples [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Simple implementation, direct spatial domain operation, suitable for non-uniform sampling if coordinates are tracked [G].
- Limitations: Can be less cache-friendly than feed-forward methods due to repeated access to input samples [G].

**When to Use:**
- Use when the output resolution is significantly higher than the input resolution, or when the kernel support is small relative to the input size [G].

**When Not to Use:**
- Avoid when the input size is very large and the kernel support is wide, as the repeated access to input samples can become a bottleneck [G].

## Background
Feed-backward reconstruction was described by Feibush in 1980 [S1, unofficial]. It is a fundamental technique in volume rendering for resampling and reconstructing volume data onto a screen-space grid.

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
