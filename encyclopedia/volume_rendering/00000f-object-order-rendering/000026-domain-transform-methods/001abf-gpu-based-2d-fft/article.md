# GPU-based 2D FFT

Also called: 2D FFT on GPU, Fast Fourier Transform on GPU, Frequency Domain Volume Rendering Transform, GPU FFT, GPU-based Fast Fourier Transform (FFT)

## Overview
GPU-based 2D FFT is a hardware-accelerated method for computing the Discrete Fourier Transform (DFT) on graphics processing units, specifically adapted for the domain-transform pipeline of volume rendering [S1, S2, unofficial]. It solves the computational bottleneck of transforming 2D projection slices between the frequency and spatial domains by exploiting the separability of the 2D DFT into two 1D passes [S2, unofficial]. It sets itself apart from CPU-based FFTs by utilizing texture memory and fragment shaders for parallel butterfly operations [S1, S2, unofficial].

## How it works
The method computes the 2D DFT of a data block of size $N \times N$ (where $N$ is typically 256 or 512) by performing two identical 1D FFT passes: one for rows and one for columns [S2, unofficial]. Each 1D pass consists of a scrambling step followed by $\log_2 N$ butterfly passes [S1, S2, unofficial].

**Inputs and Outputs:**
*   **Input:** A 2D texture containing the complex data to be transformed (e.g., the extracted projection slice from the frequency domain) [S2, unofficial].
*   **Output:** A 2D texture containing the transformed complex data (e.g., the spatial domain image for compositing) [S2, unofficial].

**Data Structures:**
1.  **Ping-Pong Buffers:** Two 2D textures of size $N \times N$ are used to store intermediate results [S1, S2, unofficial]. One acts as the source texture, and the other as the rendering target (framebuffer attachment) [S1, S2, unofficial]. They swap roles after each butterfly pass [S1, S2, unofficial].
2.  **Scramble Lookup Texture:** A 1D or 2D texture storing the bit-reversal permutation indices [S1, S2, unofficial]. It maps an input index $x$ to its bit-reversed counterpart $y$ [S1, S2, unofficial].
3.  **Butterfly Lookup Texture:** A texture storing the parameters for the butterfly operation [S1, unofficial]. For each fragment (pixel), it provides the memory address (or texture coordinate) of sample $a$, the memory address of sample $b$, and the complex value $w$, which is the principal $N$-th root of unity [S1, S2, unofficial].

**Algorithm Steps:**

1.  **Scrambling Pass:**
    *   Bind the input texture as source and a ping-pong buffer as target [S1, S2, unofficial].
    *   For each pixel at position $x$, fetch the scrambled position $y$ from the **Scramble Lookup Texture** [S1, S2, unofficial].
    *   Compute $y = \text{bit\_reverse}(x)$ [S2, unofficial].
    *   Sample the input texture at $x$ and write it to position $y$ in the target buffer [S1, S2, unofficial].
    *   *Note:* This step reorders the data into bit-reversed order, preparing it for the iterative butterfly structure [S2, unofficial].

2.  **Butterfly Passes (for one dimension):**
    *   Perform $\log_2 N$ passes [S1, S2, unofficial].
    *   Bind the current result buffer as source and the other ping-pong buffer as target [S1, S2, unofficial].
    *   For each fragment (pixel) in the target:
        *   Fetch addresses $a$ and $b$, and the complex weight $w$ from the **Butterfly Lookup Texture** [S1, S2, unofficial].
        *   Sample the source texture at $a$ to get value $A$.
        *   Sample the source texture at $b$ to get value $B$.
        *   Compute the butterfly operation:
            $$
            \begin{aligned}
            A' &= A + B \cdot w \\
            B' &= A - B \cdot w
            \end{aligned}
            $$
        *   Write $A'$ and $B'$ to their respective positions in the target buffer [S1, S2, unofficial].
    *   Swap the source and target buffers for the next pass [S1, S2, unofficial].

3.  **Dimension Swap:**
    *   After completing the $\log_2 N$ butterfly passes for the first dimension (e.g., rows), the data is now transformed in that dimension [S1, S2, unofficial].
    *   Analogously perform the same operations (scramble and butterfly passes) for the other dimension (columns) [S1, S2, unofficial].

4.  **Inverse Transform (IFFT):**
    *   The inverse transform uses the same scrambling and butterfly structure [S2, unofficial].
    *   In the final butterfly pass of the second dimension, scale the output values by $N^{-1}$ to normalize the result [S1, S2, unofficial].
    *   Write the final result to the framebuffer [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gpu_fft_2d(input_texture, output_texture, N, is_inverse=False):
    """
    Performs a 2D FFT on the GPU using ping-pong buffers and lookup textures.
    
    Args:
        input_texture: 2D texture of size N x N containing complex input data.
        output_texture: 2D texture of size N x N for the final result.
        N: Size of the transform dimension (e.g., 256, 512).
        is_inverse: Boolean flag for Inverse FFT.
    """
    # Allocate two ping-pong buffers of size N x N
    buffer_a = allocate_2d_texture(N, N)
    buffer_b = allocate_2d_texture(N, N)
    
    src = input_texture
    dst = buffer_a
    
    # Pass 1: Transform Rows
    # Step 1: Scrambling
    dst = scramble_pass(src, N)
    src = dst
    dst = buffer_b
    
    # Step 2: Butterfly Passes for Rows
    for k in range(log2(N)):
        w = get_butterfly_weight(N, k, is_inverse)
        src, dst = ping_pong_butterfly_pass(src, dst, N, w)
        
    # Pass 2: Transform Columns
    # The same 1D FFT kernel is applied to columns.
    # Step 1: Scrambling
    dst = scramble_pass(src, N)
    src = dst
    dst = buffer_b
    
    # Step 2: Butterfly Passes for Columns
    for k in range(log2(N)):
        w = get_butterfly_weight(N, k, is_inverse)
        src, dst = ping_pong_butterfly_pass(src, dst, N, w)
        
    # Final Scaling for Inverse Transform
    if is_inverse:
        dst = scale_texture(dst, N, N, 1.0 / N)
        
    # Copy result to output
    output_texture = copy_texture(dst)

def scramble_pass(src_texture, N):
    """
    Reorders data by bit-reversal permutation.
    """
    dst_texture = allocate_2d_texture(N, N)
    # Fragment shader logic:
    # x = current pixel coordinate
    # y = bit_reverse(x)
    # val = sample(src_texture, x)
    # write val to dst_texture at y
    return dst_texture

def ping_pong_butterfly_pass(src_texture, dst_texture, N, w):
    """
    Performs one butterfly pass.
    """
    # Fragment shader logic:
    # Fetch a, b, w from butterfly lookup texture based on fragment index
    # A = sample(src_texture, a)
    # B = sample(src_texture, b)
    # A_new = A + B * w
    # B_new = A - B * w
    # Write A_new, B_new to dst_texture at positions a, b
    # Swap roles for next iteration
    return dst_texture, src_texture

def get_butterfly_weight(N, k, is_inverse):
    """
    Returns the principal N-th root of unity for stage k.
    """
    exponent = -2 * pi * k / N if not is_inverse else 2 * pi * k / N
    return complex(cos(exponent), sin(exponent))

def allocate_2d_texture(width, height):
    """Allocates a 2D texture of given dimensions."""
    return new_texture(width, height)

def copy_texture(src):
    """Copies texture data to a new texture or framebuffer."""
    return src

def scale_texture(src, width, height, factor):
    """Multiplies every element in the texture by factor."""
    for y in range(height):
        for x in range(width):
            src[y, x] *= factor
    return src
```

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** The DFT assumes periodic boundary conditions [S1, unofficial]. Absence of zero-padding in the spatial domain causes copies of the dataset to appear in the rendered image [S1, unofficial].

## Subtopics

### [GPU-based FFT Convolution](4274-gpu-based-fft-convolution.md)
GPU-based FFT Convolution is a method to calculate the matrix-vector product $Hx$ (convolution) efficiently using CUDA's FFT library to reduce computational cost. It is computationally economical for large kernels where the cost of the Fast Fourier Transform (FFT) is outweighed by the reduction in arithmetic operations compared to direct spatial convolution. It builds on [GPU-based 2D FFT](841-gpu-based-2d-fft.md).

### [Split-Stream-FFT](6639-split-stream-fft.md)
Split-Stream-FFT is a GPU-optimized implementation of the Fast Fourier Transform designed to map the recursive structure of the FFT to the stream architecture of graphics hardware. It solves the problem of efficiently computing the Discrete Fourier Transform (DFT) for volume rendering and other DSP applications by leveraging the parallel processing capabilities of commodity GPUs. It sets itself apart from sibling methods like [GPU-based FFT Convolution](4274-gpu-based-fft-convolution.md) by focusing on the raw transform computation using a Decimation-in-Frequency (DIF) approach, rather than the convolution pipeline itself.

### Inverse 2D FFT (Butterfly)
An inverse 2D FFT implementation based on Hart et al., exploiting separability by transforming rows then columns, using scrambling and butterfly passes [S2, unofficial]. It splits the 2D transform into two passes: first transform rows (stack of independent 1D arrays), then transform columns [S2, unofficial]. Each pass consists of a reordering step (scrambling) followed by $\log_2(N)$ butterfly passes, where $N$ is the number of rows or columns [S2, unofficial]. Scrambling swaps samples at position $x$ and $y$, where $y$ is the bit-reversal of $x$ ($y = \text{bit\_reverse}(x)$), using a pre-computed scramble lookup texture for efficiency [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** For a $512 \times 512$ transform, the IFFT takes approximately 35-40 ms on the hardware tested in the source papers [S1, unofficial].
*   **Strengths:** High parallelism, efficient use of GPU texture units, suitable for real-time or near-real-time volume rendering when combined with efficient slice extraction [S1, S2, unofficial].
*   **Limitations:** Fixed resolution (power of 2), requires zero-padding to avoid artifacts [S1, unofficial].

## Background
The method is based on the work of Hart et al. (Engel 2003), who implemented a GPU-based FFT for volume rendering [S1, S2, unofficial]. It was further detailed and applied to frequency domain volume rendering by Viola et al. in 2004 [S1, S2, unofficial]. The approach leverages the separability of the 2D DFT and the parallel architecture of GPUs to accelerate the domain-transform pipeline [S1, S2, unofficial].

## Sources
[S1] PDF GPU-based Frequency Domain Volume Rendering - TU Wien, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Hardware-Accelerated Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/viola-2004-har2/viola-2004-har2-Paper.pdf, fetched 2026-10-27 (unofficial)
