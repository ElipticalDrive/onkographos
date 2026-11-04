# Feed-Backward Reconstruction

Also called: Gather-based reconstruction, Ray-casting approach

## Overview
Feed-backward reconstruction is a spatial-domain convolution method that computes each output sample by gathering weighted contributions from neighboring input samples. It solves the problem of resampling discrete volume data onto a finer or different grid by centering the reconstruction kernel at the output location and summing the input values weighted by the kernel function. It is distinguished from its sibling [Composition via Convolution (Power Form)](1591-composition-via-convolution-power-form.md) by its explicit spatial gathering loop structure, which accesses each output sample once and each input sample multiple times, contrasting with feed-forward approaches that distribute input energy to multiple outputs.

## How it works
**Inputs and Outputs:**
*   **Input:** A discrete signal $f$ (volume samples), a reconstruction kernel $h$, and a set of output positions.
*   **Output:** A discrete signal $g$ (resampled volume samples).

**Data Structures:**
*   `InputSamples`: Array of values $f[i]$.
*   `OutputSamples`: Array of values $g[j]$, initialized to zero.
*   `Kernel`: Function $h(\cdot)$ defining the weight based on distance.

**Numbered Steps:**
1.  Initialize all output samples $g[j]$ to $0$.
2.  Iterate through each output sample index $j$ (representing the output position).
3.  For each output sample $j$, identify the set of input samples $i$ that lie within the support (extent) of the reconstruction kernel centered at the output position.
4.  For each input sample $i$ in the support:
    *   Calculate the kernel weight $w = h(\text{distance between output position } j \text{ and input position } i)$.
    *   Accumulate the contribution: $g[j] \leftarrow g[j] + f[i] \times w$.
5.  Repeat for all output samples.

**Formulas:**
The operation is a discrete convolution sum [S1, unofficial]:
$$g(x, y, z) = \sum_{i=-\infty}^{\infty} \sum_{j=-\infty}^{\infty} \sum_{k=-\infty}^{\infty} f(i, j, k) h(x - i, y - j, z - k)$$
where $f$ is the input signal, $h$ is the filter function, and $g$ is the output signal [S1, unofficial].

**Parameters:**
*   **Kernel Extent:** Determines how many input samples are accessed per output sample. Common kernels include the triangular pulse (linear interpolation) [S1, unofficial].
*   **Sampling Rate:** The ratio of output samples to input samples affects the total number of accesses.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Input samples near the edge of the volume may have fewer neighbors within the kernel support. The implementation must handle indices outside the input array bounds (e.g., by clamping, mirroring, or assuming zero) [G].
*   **Precision:** Accumulating many small weights can lead to floating-point precision loss if not managed carefully, though standard double precision is usually sufficient for typical volume sizes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def feed_backward_reconstruction(input_samples, kernel_func, output_positions):
    """
    Performs feed-backward reconstruction.
    
    Args:
        input_samples: List or array of input volume values.
        kernel_func: Function that takes a distance and returns a weight.
        output_positions: List of positions where output samples are desired.
        
    Returns:
        List of output volume values.
    """
    num_outputs = len(output_positions)
    output_samples = [0.0] * num_outputs
    
    # Pre-calculate input positions if not provided, assuming uniform spacing for simplicity
    # In a real implementation, input_positions would be passed or derived from metadata
    # Here we assume input_samples[i] is at position i for the sake of the algorithm structure
    # relative to output_positions which are also in the same coordinate space.
    
    for out_idx, out_pos in enumerate(output_positions):
        current_sum = 0.0
        
        # Iterate over all input samples to find those under the kernel
        # Optimization: In practice, only iterate over input indices within kernel support
        for in_idx, in_val in enumerate(input_samples):
            # Calculate distance between output position and input position
            # Assuming input at index i is at position i (normalized)
            dist = abs(out_pos - in_idx)
            
            # Check if input sample is within kernel support
            # The kernel support is defined by where kernel_func(dist) > 0
            weight = kernel_func(dist)
            
            if weight != 0:
                current_sum += in_val * weight
                
        output_samples[out_idx] = current_sum
        
    return output_samples
```

## Subtopics

### Spatial Convolution
Spatial convolution is the operation of convolving discrete samples with a reconstruction kernel in the spatial domain, as opposed to the frequency domain. It is preferred over frequency domain multiplication for discrete kernels of limited extent because transforming back and forth is more compute-intensive than direct spatial convolution [S1, unofficial]. The mathematical definition in 3D is:
$$g(x, y, z) = \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} f(u, v, w) h(x - u, y - v, z - w) du dv dw$$
[S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(N_{out} \times N_{kernel})$, where $N_{out}$ is the number of output samples and $N_{kernel}$ is the number of input samples within the kernel support.
*   **Memory:** $O(N_{out})$ for the output buffer, plus $O(N_{in})$ for the input buffer.

**Measured Performance:**
In a specific example with 4 input samples, 13 output samples, and linear reconstruction (kernel extent covering 2 input samples per output), feed-backward reconstruction results in 39 sample accesses (2 input accesses per output $\times$ 13 outputs + 13 output writes) [S1, unofficial]. This contrasts with feed-forward reconstruction, which would result in 24 sample accesses (5 output accesses per input $\times$ 4 inputs + 4 input reads) [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Intuitive mapping to the output grid; efficient for generating high-resolution outputs from lower-resolution inputs when the kernel is small.
*   **Limitations:** High memory bandwidth usage for input samples, as each input sample is read multiple times (once for each output sample it influences).

**When to Use:**
Use when the output grid is significantly finer than the input grid, or when the kernel support is small. It is the standard approach in many volume renderers for resampling [S1, unofficial].

## Background
Feed-backward reconstruction was described by Feibush in 1980 [S1, unofficial]. It is a fundamental technique in digital signal processing and computer graphics for resampling data. The method is based on the general theory of convolution, as detailed by Oppenheim in 1975 [S1, unofficial]. It is widely used in volume rendering algorithms that operate in the spatial domain, particularly those using small kernels [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
