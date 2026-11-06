# GPU-based 2D FFT

Also called: 2D FFT on GPU, Fast Fourier Transform on GPU, Frequency Domain Volume Rendering Transform, GPU FFT, GPU-based Fast Fourier Transform (FFT)

## Overview
A GPU-based implementation of the 2D Fast Fourier Transform (FFT) used for volume rendering, specifically designed to exploit the separability of the Fourier transform by processing rows and columns independently [S1, unofficial][S2, unofficial]. It solves the computational bottleneck of frequency-domain volume rendering by mapping the butterfly algorithm to GPU fragment shaders, using texture lookups for address calculation and twiddle factors [S1, unofficial][S2, unofficial]. It sets itself apart from CPU-based FFTs by leveraging parallel fragment processing and hardware texture units for data reordering and arithmetic, and from other domain-transform methods like the Hartley Transform by using complex arithmetic and standard bit-reversal scrambling [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 2D texture containing complex data (real and imaginary parts) of size $N \times N$ [S1, unofficial][S2, unofficial].
*   Output: A 2D texture containing the transformed complex data of size $N \times N$ [S1, unofficial][S2, unofficial].

**Data Structures:**
1.  **Ping-Pong Buffers:** Two 2D textures of size $N \times N$ used to store intermediate results [S1, unofficial][S2, unofficial]. One acts as the source texture, the other as the rendering target [S1, unofficial][S2, unofficial].
2.  **Scramble Lookup Texture:** A pre-computed texture storing the bit-reversal permutation indices [S1, unofficial][S2, unofficial]. For an index $x$, it provides the swapped index $y = \text{bit\_reverse}(x)$ [S1, unofficial][S2, unofficial].
3.  **Butterfly Lookup Texture:** A pre-computed texture storing the addresses of the two contributing samples ($a$ and $b$) and the principal $N$-th root of unity ($w$) for each butterfly operation [S1, unofficial][S2, unofficial].

**Parameters:**
*   $N$: The size of the transform dimension (e.g., 256 or 512) [S1, unofficial][S2, unofficial].

**Algorithm Steps:**
The 2D FFT is computed in two identical passes: one for the rows and one for the columns [S1, unofficial][S2, unofficial]. Each pass consists of a scrambling phase followed by $\log_2 N$ butterfly phases [S1, unofficial][S2, unofficial].

1.  **Scrambling Pass:**
    *   Reorder the input data by swapping samples at position $x$ with samples at position $y$, where $y$ is the bit-reversal of $x$ [S1, unofficial][S2, unofficial].
    *   This is implemented by binding the input texture as a source and a buffer as a target [S1, unofficial][S2, unofficial]. Each fragment shader invocation reads the coordinate $x$, fetches $y$ from the scramble lookup texture, and writes the value from $x$ to $y$ [S1, unofficial][S2, unofficial].
    *   Formula: $y = \text{bit\_reverse}(x)$ [S2, unofficial].

2.  **Butterfly Passes:**
    *   Perform $\log_2 N$ passes [S1, unofficial][S2, unofficial]. In each pass, use ping-pong buffering: bind one buffer as the source texture and the other as the rendering target [S1, unofficial][S2, unofficial].
    *   In each butterfly pass, the fragment shader processes a fragment corresponding to a specific output location [S1, unofficial].
    *   Fetch from the butterfly lookup texture to determine the addresses of two contributing samples, $a$ and $b$ [S1, unofficial][S2, unofficial], and the principal $N$-th root of unity, $w$ [S1, unofficial][S2, unofficial].
    *   Perform complex multiplication and two additions using the fetched values and addresses [S1, unofficial][S2, unofficial].

3.  **Second Dimension:**
    *   Analogously perform the same scrambling and butterfly passes for the other dimension [S1, unofficial][S2, unofficial].

4.  **Inverse Transform:**
    *   For the inverse 2D FFT, the process is identical, but in the final pass, the values are scaled down by the factor $N^{-1}$ [S1, unofficial][S2, unofficial].
    *   The output is written to the framebuffer [S1, unofficial][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def gpu_fft_2d(input_data, output_data, is_inverse=False):
    """
    Performs a 2D FFT on the GPU using ping-pong buffering and lookup textures.
    input_data: 2D list of complex numbers size N x N
    output_data: 2D list of complex numbers size N x N
    """
    N = len(input_data)
    LOG_N = int(math.log2(N))
    # Initialize ping-pong buffers
    buf_a = [row[:] for row in input_data]
    buf_b = [[0j]*N for _ in range(N)]

    # Pass 1: Rows
    buf_a = scramble_pass(buf_a)
    current_src, current_dst = buf_b, buf_a
    for stage in range(LOG_N):
        current_dst = butterfly_pass(current_src, current_dst, stage, N)
        current_src, current_dst = current_dst, current_src
    row_result = current_src

    # Pass 2: Columns
    col_result = scramble_pass(transpose(row_result))
    current_src, current_dst = buf_b, buf_a
    for stage in range(LOG_N):
        current_dst = butterfly_pass(current_src, current_dst, stage, N)
        current_src, current_dst = current_dst, current_src
    final_result = transpose(current_src)

    if is_inverse:
        for i in range(N):
            for j in range(N):
                final_result[i][j] /= N
    return final_result

def scramble_pass(data):
    """
    Reorders data by swapping x with bit_reverse(x).
    """
    N = len(data)
    result = [[0j]*N for _ in range(N)]
    for y in range(N):
        for x in range(N):
            idx = y * N + x
            swapped_idx = bit_reverse(idx, int(math.log2(N)))
            result[swapped_idx // N][swapped_idx % N] = data[y][x]
    return result

def butterfly_pass(src, dst, stage, N):
    """
    Performs one stage of the FFT butterfly operations.
    """
    for y in range(N):
        for x in range(N):
            # Fetch butterfly parameters from lookup texture
            addr_a, addr_b, w = fetch_butterfly_lookup(stage, x, y)
            val_a = src[addr_a // N][addr_a % N]
            val_b = src[addr_b // N][addr_b % N]
            # Complex multiplication: val_b * w
            prod_real = val_b.real * w.real - val_b.imag * w.imag
            prod_imag = val_b.real * w.imag + val_b.imag * w.real
            # Complex addition: val_a + prod
            dst[y][x] = complex(val_a.real + prod_real, val_a.imag + prod_imag)
    return dst

def transpose(matrix):
    N = len(matrix)
    return [[matrix[j][i] for j in range(N)] for i in range(N)]

def bit_reverse(x, bits):
    result = 0
    for _ in range(bits):
        result = (result << 1) | (x & 1)
        x >>= 1
    return result

def fetch_butterfly_lookup(stage, x, y):
    # Returns (addr_a, addr_b, w) from pre-computed texture
    # Implementation details depend on specific lookup texture layout
    return (0, 0, complex(1, 0)) # Placeholder for lookup fetch
```

**Edge Cases and Pitfalls:**
*   **Zero-Padding:** Absence of zero-padding in the spatial domain causes copies of the dataset to appear in the rendered image due to periodicity assumptions in the FFT [S1, unofficial].
*   **Precision:** Floating-point precision in texture lookups and arithmetic may accumulate errors, especially for large $N$ [G].
*   **Boundary Conditions:** The bit-reversal and butterfly indices must be correctly calculated for all $x$ in $[0, N-1]$ [S1, unofficial][S2, unofficial].

## Subtopics

### Inverse 2D FFT (Butterfly)
An inverse 2D FFT implementation based on Hart et al., exploiting separability by transforming rows then columns, using scrambling and butterfly passes [S2, unofficial]. It is chosen when converting frequency domain data back to spatial domain for rendering [S2, unofficial]. It differs from the forward transform only by the final scaling factor $N^{-1}$ [S1, unofficial][S2, unofficial].

### [GPU-based FFT Convolution](4274-gpu-based-fft-convolution.md)
A method to calculate the matrix-vector product $Hx$ (convolution) efficiently using CUDA's FFT library to reduce computational cost. It is chosen for filtering operations in volume rendering. It differs from the standard 2D FFT by including padding, point-wise multiplication in frequency domain, and inverse FFT steps.

### [Split-Stream-FFT](6639-split-stream-fft.md)
A novel implementation of the Fast Fourier Transform designed to map the recursive structure of the FFT to the stream architecture of a GPU efficiently. It is based on the Decimation-in-Frequency (DIF) approach. Not given in the sources.

## Operating characteristics
*   **Time Complexity:** $O(N^2 \log N)$ for a 2D transform of size $N \times N$ [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** $O(N^2)$ for storing the input, output, and intermediate buffers [S1, unofficial][S2, unofficial].
*   **Measured Performance:** For a $256 \times 256$ transform, the 2D FFT takes 153 units [S1, unofficial]. For $512 \times 512$, it takes 35 units [S1, unofficial]. The source states "TL", "TC", "IFFT", "2D" columns [S1, unofficial]. The units are not given in the sources.
*   **Strengths:** High parallelism on GPU, efficient use of texture units for data access patterns [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires pre-computed lookup textures, which consume memory [S1, unofficial][S2, unofficial]. Precision issues may arise for large $N$ [G].
*   **When to use:** Not given in the sources.
*   **When not to use:** When memory is extremely constrained (consider Hartley Transform [S1, unofficial][S2, unofficial]) or when the data size is small enough that CPU FFT is sufficient [G].

## Background
The method is based on the implementation by Hart et al. [Engel 2003] [S1, unofficial][S2, unofficial]. It was developed to accelerate frequency domain volume rendering on GPUs [S1, unofficial][S2, unofficial]. The approach leverages the separability of the 2D FFT and the parallel processing capabilities of GPUs [S1, unofficial][S2, unofficial].

## Sources
[S1] GPU-based Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S2] Hardware-Accelerated Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/viola-2004-har2/viola-2004-har2-Paper.pdf, fetched 2026-10-27 (unofficial)
[S3] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
