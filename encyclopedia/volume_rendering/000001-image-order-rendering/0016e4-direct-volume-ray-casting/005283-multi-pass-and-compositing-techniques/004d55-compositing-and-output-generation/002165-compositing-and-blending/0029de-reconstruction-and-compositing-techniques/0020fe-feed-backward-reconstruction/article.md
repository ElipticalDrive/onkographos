# Feed-Backward Reconstruction

Also called: Gather-based reconstruction, Ray-casting approach

## Overview
Feed-Backward Reconstruction is a volume rendering technique that computes each output pixel (or sample) by gathering contributions from neighboring input volume samples using a reconstruction kernel [S1, unofficial]. It solves the problem of resampling discrete volume data onto a continuous image grid by treating convolution as a "gather" operation, where the kernel is centered at the output location [S1, unofficial]. It is distinguished from its sibling [Feed-Forward Reconstruction](2514-feed-forward-reconstruction.md) by its access pattern: it accesses each output sample once and each input sample multiple times, whereas feed-forward spreads energy from each input to multiple outputs [S1, unofficial]. This approach aligns naturally with image-order ray casting, where pixels are generated sequentially [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of discrete volume samples $X_j$ (where $j$ indexes the input grid positions) and a reconstruction kernel $K(u)$ defined by its support width.
- **Output:** A set of continuous image samples $A_i$ (where $i$ indexes the pixel centers or ray sample positions).

**Data Structures:**
- **Input Samples:** An array or texture storing scalar or vector values $X_j$.
- **Kernel:** A function or lookup table defining weights $K(u)$ based on distance $u$ from the kernel center. Common kernels include linear (triangular pulse) or B-splines [S1, unofficial].
- **Output Buffer:** An array initialized to zero for each output sample $A_i$.

**Algorithm Steps:**
1. Initialize all output samples $A_i = 0$.
2. For each output sample $i$ (from front to back along the ray, or in raster order):
    a. Determine the spatial position $p_i$ of the output sample.
    b. Identify all input samples $X_j$ whose positions $p_j$ lie within the support of the kernel centered at $p_i$.
    c. For each such input sample $j$:
        i. Calculate the distance $u = p_i - p_j$.
        ii. Compute the kernel weight $w = K(u)$.
        iii. Accumulate the contribution: $A_i \leftarrow A_i + X_j \cdot w$.
3. Composite the final accumulated color $A_i$ over the background.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def feed_backward_reconstruction(input_samples, kernel, output_positions):
    """
    input_samples: list of tuples (position, value)
    kernel: function K(u) returning weight for distance u
    output_positions: list of positions for output samples
    returns: list of output values
    """
    output_values = [0.0] * len(output_positions)
    
    # Pre-calculate kernel support radius to optimize search
    # Not given in sources, assumed to be derived from kernel definition [G]
    support_radius = get_kernel_support(kernel) 
    
    for i, pos_out in enumerate(output_positions):
        acc = 0.0
        for pos_in, val_in in input_samples:
            dist = abs(pos_out - pos_in)
            if dist <= support_radius:
                weight = kernel(dist)
                acc += val_in * weight
        output_values[i] = acc
        
    return output_values
```

**Parameters:**
- **Kernel Type:** Linear (triangular), B-spline, etc. [S1, unofficial].
- **Sampling Rate:** The ratio of output samples to input samples affects the number of accesses [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Kernel Support:** If the kernel support is not correctly bounded, irrelevant samples may be included, or relevant ones excluded, causing artifacts [S1, unofficial].
- **Precision:** Accumulating many small weights can lead to floating-point precision issues if not managed, though standard float32 is typically sufficient for typical volume rendering scales [G].
- **Boundary Conditions:** Input samples near the volume boundary may have partial kernel support outside the volume; these must be handled (e.g., by clamping or zero-padding) [G].

## Subtopics

### [Spatial Convolution](2513-spatial-convolution.md)
Spatial convolution is a neighborhood-based operation where output values are calculated as a weighted average of neighboring input samples using a filter kernel. In volume rendering, it is used to convolve discrete samples with a reconstruction kernel in the spatial domain. It is distinguished from frequency-domain methods (like FFT-based convolution) by operating directly on spatial data, which is preferred for discrete kernels of limited extent because transforming to and from the frequency domain is more compute-intensive than direct spatial convolution. Feed-backward reconstruction is a specific implementation strategy for spatial convolution.

### [Area-based Pixel Kernel](14355-area-based-pixel-kernel.md)
The Area-based Pixel Kernel is a reconstruction kernel used in feed-backward reconstruction that assigns pixel contributions proportional to the geometric area of intersection between a pixel and a thick projection line. It solves the artifact issues inherent in Dirac and B-Spline kernels, which leave unreconstructed pixels during subsampling, and linear kernels, which fail to fully exploit available data during oversampling. It is distinguished from its sibling [Spatial Convolution](2513-spatial-convolution.md) by its specific geometric interpretation of projection thickness and its optimization via rotation-based approximation.

### Feed-Forward Reconstruction
Feed-Forward Reconstruction is a reconstruction method that views convolution as distributing contributions from a given input sample to its neighboring output samples [S1, unofficial]. It centers the reconstruction kernel at an input value and spreads the energy to each output lying under the kernel [S1, unofficial].
**Steps:**
1. Zero all Output samples.
2. For each input sample $J$:
    a. For each output sample $I$ under Kernel centered at $J$:
        i. Output sample $I \leftarrow$ Output sample $I +$ Input sample $J \times$ Kernel value at $I$.
**Comparison:** Accesses each input sample once and each output sample many times [S1, unofficial]. In an example with 4 input samples and 13 output samples using linear reconstruction, it results in 24 sample accesses (5 output accesses per input $\times$ 4 inputs + 4 input reads) [S1, unofficial]. Feed-backward and feed-forward methods produce identical results but differ in memory access patterns [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Time:** $O(N_{out} \cdot N_{kernel})$, where $N_{out}$ is the number of output samples and $N_{kernel}$ is the number of input samples within the kernel support [S1, unofficial].
- **Memory:** $O(N_{in} + N_{out})$ for storing input and output samples [S1, unofficial].

**Measured Performance:**
- In a specific example with 4 input samples, 13 output samples, and linear reconstruction, feed-backward reconstruction required 39 sample accesses (2 input accesses per output $\times$ 13 outputs + 13 output writes) [S1, unofficial].
- Feed-forward reconstruction required 24 sample accesses in the same scenario [S1, unofficial].
- The total number of accesses depends on the kernel extent and the number of intermediate samples [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Natural fit for image-order rendering; efficient cache usage if input samples are accessed sequentially for each output [G].
- **Limitations:** May require more memory bandwidth if the kernel support is large, as each output sample triggers multiple input reads [S1, unofficial].

**When to Use:**
- Use when the rendering pipeline is image-order (ray casting) and the kernel support is small to moderate [S1, unofficial].
- Avoid when the input data is extremely large and the kernel support is wide, making repeated input accesses prohibitive [G].

## Background
Feed-backward reconstruction was described by Feibush in 1980 [S1, unofficial]. It is based on the principle that convolution can be viewed as gathering contributions from neighboring input samples for each output sample [S1, unofficial]. This method is widely used in direct volume ray casting for its simplicity and alignment with pixel-centric rendering loops [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
