# Feed-Backward Reconstruction

Also called: Gather-based reconstruction, Ray-casting approach

## Overview
Feed-Backward Reconstruction is a method for resampling discrete input data into a continuous output signal by treating convolution as a gathering operation [S1, unofficial]. It solves the problem of reconstructing a continuous signal from discrete samples by calculating each output value individually, centering a reconstruction kernel at the output position and summing the weighted contributions of neighboring input samples [S1, unofficial]. It is distinguished from its sibling [Feed-Forward Reconstruction](2513-spatial-convolution.md) by its access pattern: it accesses each output sample once and each input sample multiple times, whereas feed-forward spreads each input sample to multiple outputs [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of discrete input samples $X = \{x_j\}$ [S1, unofficial].
- Input: A reconstruction kernel $K(u)$, defined by its support width and shape (e.g., triangular for linear interpolation) [S1, unofficial].
- Input: A set of output positions $Y = \{y_i\}$ [S1, unofficial].
- Output: A set of reconstructed output values $A = \{a_i\}$ [S1, unofficial].

**Data Structures:**
- `input_samples`: Array of values $x_j$.
- `output_positions`: Array of coordinates $y_i$.
- `kernel`: Function or lookup table defining $K(u)$.
- `output_samples`: Array of values $a_i$, initialized to 0.

**Numbered Steps:**
1. Initialize the output array `output_samples` with zeros for all output indices $i$ [S1, unofficial].
2. Iterate through each output sample index $i$ [S1, unofficial]:
    a. Set the current output value $a_i = 0$ [S1, unofficial].
    b. Identify all input samples $x_j$ that lie within the support of the reconstruction kernel centered at $y_i$ [S1, unofficial].
    c. For each input sample $j$ in the support:
        i. Calculate the relative distance $u = y_i - y_j$ [S1, unofficial].
        ii. Evaluate the kernel weight $w = K(u)$ [S1, unofficial].
        iii. Accumulate the contribution: $a_i \leftarrow a_i + x_j \cdot w$ [S1, unofficial].
3. The final array `output_samples` contains the reconstructed values [S1, unofficial].

**Formulas:**
The value of the $i$-th output sample $a_i$ is computed as:
$$
a_i = \sum_{j \in J_i} x_j \cdot K(y_i - y_j)
$$
where:
- $x_j$ is the value of the $j$-th input sample [S1, unofficial].
- $K(\cdot)$ is the reconstruction kernel function [S1, unofficial].
- $y_i$ is the position of the $i$-th output sample [S1, unofficial].
- $y_j$ is the position of the $j$-th input sample [S1, unofficial].
- $J_i$ is the set of input indices $j$ such that $|y_i - y_j|$ is within the support of $K$ [S1, unofficial].

**Parameters:**
- Kernel support width: Determines the number of input samples $j$ to consider for each output $i$. For a triangular pulse (linear interpolation), the support is typically 2 units (covering immediate neighbors) [S1, unofficial].
- Output sampling rate: The density of output positions $y_i$ relative to input positions $y_j$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Kernel Support Boundaries:** If an output sample is positioned such that no input samples fall within the kernel support, the output value will be 0 [S1, unofficial].
- **Precision:** Accumulating many small weights can lead to floating-point precision issues. Using double precision for the accumulator is recommended if the number of contributing samples is large [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def triangular_kernel(u):
    """Triangular kernel K(u) = max(0, 1 - |u|) for linear interpolation."""
    return max(0.0, 1.0 - abs(u))

def feed_backward_reconstruction(input_samples, input_positions, output_positions):
    """
    Reconstructs output samples by gathering contributions from input samples.
    Implements the feed-backward method using a triangular kernel (linear interpolation).
    """
    num_outputs = len(output_positions)
    num_inputs = len(input_samples)
    output_samples = [0.0] * num_outputs

    for i in range(num_outputs):
        y_i = output_positions[i]
        current_output = 0.0

        for j in range(num_inputs):
            y_j = input_positions[j]
            distance = abs(y_i - y_j)

            if distance <= 1.0:  # Support radius for triangular kernel
                weight = triangular_kernel(distance)
                current_output += input_samples[j] * weight

        output_samples[i] = current_output

    return output_samples
```

## Subtopics

### Feed-Forward Reconstruction
Feed-Forward Reconstruction is the dual approach where convolution is viewed as distributing contributions from each input sample to its neighboring output samples [S1, unofficial]. It centers the reconstruction kernel at an input value and spreads the energy to each output lying under the kernel [S1, unofficial]. It is distinguished from feed-backward by its access pattern: it accesses each input sample once and each output sample many times [S1, unofficial]. For example, with 4 input samples and 13 output samples using linear reconstruction, feed-forward results in 24 sample accesses (5 output accesses per input * 4 inputs + 4 input reads), whereas feed-backward results in 39 sample accesses (2 input accesses per output * 13 outputs + 13 output writes) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** In the example of 4 input samples and 13 output samples with linear reconstruction, feed-backward requires 39 sample accesses (2 input reads per output * 13 outputs + 13 output writes) [S1, unofficial]. Feed-forward requires 24 sample accesses (5 output writes per input * 4 inputs + 4 input reads) [S1, unofficial].
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
Feed-Backward Reconstruction is a fundamental technique in signal processing and computer graphics for resampling data. It is based on the principle of convolution, where the output is the weighted sum of input samples. The method is referenced in volume rendering literature as [Feibush 80] [S1, unofficial]. It is widely used in image processing, texture mapping, and volume rendering pipelines where high-quality reconstruction is required.

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
