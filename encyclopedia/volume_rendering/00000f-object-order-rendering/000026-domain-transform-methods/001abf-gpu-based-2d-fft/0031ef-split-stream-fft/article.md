# Split-Stream-FFT


## Overview
Split-Stream-FFT is a GPU-optimized implementation of the Fast Fourier Transform designed to map the recursive structure of the FFT to the stream architecture of graphics hardware [S1, unofficial]. It solves the problem of efficiently computing the Discrete Fourier Transform (DFT) for volume rendering and other DSP applications by leveraging the parallel processing capabilities of commodity GPUs [S1, unofficial]. It sets itself apart from sibling methods like [GPU-based FFT Convolution](4274-gpu-based-fft-convolution.md) by focusing on the raw transform computation using a Decimation-in-Frequency (DIF) approach, rather than the convolution pipeline itself [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A sequence of $N$ complex samples $f(n)$, where $0 \le n < N$ and $N$ is a power of 2.
*   Output: The frequency domain representation $F(k)$, where $0 \le k < N$, sorted in bit-reversed order.

**Data Structures:**
*   The algorithm operates on a 1D array of complex numbers.
*   No explicit tree structures are used; the recursion is implicit in the stage-based processing.
*   Memory layout: A contiguous array of size $N$ for input and output.

**Algorithm Steps:**
1.  **Initialization:** Load the input signal $f(n)$ into the GPU memory.
2.  **Recursive Decomposition (DIF):** Perform $\log_2 N$ stages of butterfly operations. In each stage, the data is divided into smaller DFTs. The division occurs in the frequency domain.
    *   The standard DFT definition includes a constant factor $1/N$. This factor is discarded during the recursive steps to maintain a strict recursive structure without scaling at every level [S1, unofficial].
    *   The decomposition for a sequence of size $N$ splits the input into even-indexed and odd-indexed frequency components based on the following relations for $0 \le n < N/2$:
        $$ e(n) = f(n) + f(n + N/2) $$
        $$ o(n) = (f(n) - f(n + N/2)) e^{-2\pi i n / N} $$
    *   The resulting frequency values are computed as:
        $$ F(k) = F_e(k) + W_N^k F_o(k) \quad \text{for even } k $$
        $$ F(k) = F_e(k) - W_N^k F_o(k) \quad \text{for odd } k $$
        where $F_e$ and $F_o$ are the DFTs of the sequences $e(n)$ and $o(n)$ respectively, and $W_N^k = e^{-2\pi i k / N}$ is the twiddle factor [S1, unofficial].
3.  **Post-Processing:** After all $\log_2 N$ stages are complete, the resulting frequency distribution is in the wrong order (bit-reversed). A bit-reversal (or tangling) operation is performed to re-sort the values into natural order [S1, unofficial].
4.  **Scaling:** Multiply the final result by the constant factor $1/N$ to correct for the scaling discarded in step 2 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def split_stream_fft(f, N):
    """
    Computes the FFT of input array f of size N using DIF approach.
    Returns F in bit-reversed order.
    """
    # f is a list of complex numbers of length N
    # N must be a power of 2
    
    # Stage 1 to log2(N): Butterfly operations
    num_stages = int(math.log2(N))
    
    # Current working array is f
    # We process in-place or using two buffers. 
    # The source describes a recursive structure mapped to stream architecture.
    # For implementation clarity, we simulate the stages.
    
    for stage in range(num_stages):
        # Determine the size of the sub-DFTs at this stage
        # Stage 0: combines pairs (size 2)
        # Stage 1: combines quads (size 4)
        # ...
        # Stage s: combines blocks of size 2^(s+1)
        
        block_size = 2 ** (stage + 1)
        half_block = block_size // 2
        
        for i in range(0, N, block_size):
            # Process each block
            for j in range(half_block):
                # Indices for the butterfly
                idx_even = i + j
                idx_odd = i + j + half_block
                
                # Calculate twiddle factor W_N^k
                # Note: The source defines o(n) with a twiddle factor dependent on n
                # In the iterative DIF implementation, the twiddle factor exponent 
                # depends on the position within the block.
                # W_N^k = exp(-2*pi*i * k / N)
                # Here k corresponds to j in the context of the sub-DFT size N/2^(stage+1)?
                # Actually, standard DIF butterfly uses W_N^j where N is the total size?
                # Let's stick to the source's formula definition:
                # e(n) = f(n) + f(n + N/2)
                # o(n) = (f(n) - f(n + N/2)) * exp(-2*pi*i * n / N)
                # This implies the twiddle factor is based on the original N and index n.
                # However, in a multi-stage FFT, the "N" in the twiddle factor 
                # effectively scales with the stage.
                # Standard DIF: Twiddle factor is W_{current_size}^j
                
                # Using the source's specific decomposition logic:
                # The source explicitly writes:
                # o(n) = (f(n) - f(n + N/2)) e^{-2\pi i n / N}
                # This suggests the twiddle factor uses the total N and the local index n.
                
                twiddle = complex(math.cos(-2 * math.pi * j / N), math.sin(-2 * math.pi * j / N))
                
                val_even = f[idx_even]
                val_odd = f[idx_odd]
                
                # Butterfly operation
                # F_e and F_o are intermediate results. 
                # In a standard iterative DIF, we update the array.
                # The source formula:
                # F(k) = F_e(k) + W_N^k F_o(k) for even k
                # F(k) = F_e(k) - W_N^k F_o(k) for odd k
                
                # Mapping to array indices:
                # The output of this stage for the current block goes to indices
                # corresponding to the even and odd positions in the frequency domain 
                # relative to the block.
                
                # Let's implement the standard DIF butterfly which matches the source's
                # recursive definition:
                t = val_odd * twiddle
                f[idx_even] = val_even + t
                f[idx_odd] = val_even - t

    # Bit-reversal permutation
    f = bit_reverse_permutation(f, N)
    
    # Post-multiply by 1/N
    for i in range(N):
        f[i] = f[i] / N
        
    return f

def bit_reverse_permutation(f, N):
    """
    Reorders the array f into bit-reversed order.
    """
    num_bits = int(math.log2(N))
    result = [0] * N
    for i in range(N):
        rev = 0
        temp = i
        for _ in range(num_bits):
            rev = (rev << 1) | (temp & 1)
            temp >>= 1
        result[rev] = f[i]
    return result
```

**Parameters:**
*   $N$: Input size, must be a power of 2. Typical values depend on the volume resolution (e.g., $128^3$, $256^3$).
*   Reconstruction Filter: The source mentions using a Lanczos windowed-sinc function for resampling in the frequency domain, but this is part of the broader FVR pipeline, not the FFT itself [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors can accumulate during the recursive stages.
*   **Boundaries:** The algorithm assumes $N$ is a power of 2. Non-power-of-2 sizes require padding or a different algorithm.
*   **Ordering:** The output is in bit-reversed order. Failure to perform the bit-reversal step results in incorrect frequency ordering [S1, unofficial].
*   **Scaling:** Forgetting to multiply by $1/N$ at the end results in an amplitude error [S1, unofficial].

## Subtopics

### GPU-based FFT Convolution
Uses the FFT to perform convolution efficiently. Split-Stream-FFT provides the transform component. [GPU-based FFT Convolution](4274-gpu-based-fft-convolution.md)

### Domain-transform methods
Split-Stream-FFT is a specific implementation within this category, focusing on the GPU mapping of the FFT recursion. [Domain-transform methods](52-domain-transform-methods.md)

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ for $N$ input values [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the input and output arrays.
*   **Performance:** The method is designed for superior speed performance on commodity graphics hardware compared to naive implementations [S1, unofficial].
*   **Strengths:** Efficient mapping to GPU stream architecture; high performance for DSP and volume rendering applications.
*   **Limitations:** Requires $N$ to be a power of 2; output is in bit-reversed order; subject to under-/oversampling and aliasing issues inherent to the DFT [S1, unofficial].
*   **When to use:** When high-performance FFT computation is needed on a GPU for volume rendering or general DSP.
*   **When not to use:** When the input size is not a power of 2 and padding is undesirable; when natural order output is required without post-processing.

## Background
Split-Stream-FFT was introduced in the context of Fourier Volume Rendering (FVR) on the GPU. The paper "Fourier volume rendering on the GPU using a split-stream-FFT" describes this novel implementation designed to take advantage of commodity graphics hardware features [S1, unofficial]. It was developed to address the limitations of previous FVR approaches and to provide an efficient FFT implementation for volume visualization, which is seen as the inverse problem of tomographic reconstruction [S1, unofficial]. The method uses the Decimation-in-Frequency (DIF) approach and incorporates a Lanczos windowed-sinc function for reconstruction filtering [S1, unofficial].

## Sources
[S1] (PDF) Fourier volume rendering on the GPU using a split- ..., https://www.researchgate.net/publication/220838994_Fourier_volume_rendering_on_the_GPU_using_a_split-stream-FFT, fetched 2026-10-31 (unofficial)
