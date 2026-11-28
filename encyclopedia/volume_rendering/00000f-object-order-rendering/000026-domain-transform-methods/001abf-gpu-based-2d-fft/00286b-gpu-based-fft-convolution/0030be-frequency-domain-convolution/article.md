# Frequency Domain Convolution
Also called: FFT Convolution, Point Spread Function (PSF) Convolution

## Overview
Frequency Domain Convolution computes the convolution of an input signal with a kernel by transforming both into the frequency domain, multiplying their transforms, and inverse-transforming the result [S2, unofficial]. It is computationally economical for large kernels compared to direct spatial summation [S2, unofficial]. Unlike direct spatial methods, it leverages the Convolution Theorem to replace nested summation with element-wise multiplication [S2, unofficial].

## How it works
**Inputs:**
- Input signal $g$ of size $N \times N$ [S2, unofficial].
- Convolution kernel $h$ of size $K_h \times K_w$ [G].

**Outputs:**
- Convolved signal of size $N \times N$ [S2, unofficial].

**Data Structures:**
- Complex-valued arrays of size $N \times N$ for frequency domain representation [G].
- Zero-padded kernel array of size $N \times N$ [S2, unofficial].

**Steps:**
1. **Zero-Pad Kernel:** Create an array of size $N \times N$ and place the kernel $h$ at the top-left corner, filling the remainder with zeros [S2, unofficial].
2. **Forward FFT:** Compute the 2D Discrete Fourier Transform (DFT) of the input signal $g$ to obtain $G_{u,v}$ [G].
3. **Forward FFT:** Compute the 2D DFT of the padded kernel to obtain $H_{u,v}$ [G].
4. **Frequency Multiplication:** Multiply the transforms element-wise to obtain $Y_{u,v} = G_{u,v} \cdot H_{u,v}$ [S2, unofficial].
5. **Inverse FFT:** Compute the 2D Inverse DFT of $Y_{u,v}$ to obtain the result in the spatial domain [G].
6. **Normalization:** Ensure the kernel sum equals 1 to conserve energy [S1, unofficial]. (Scaling instruction not given in the sources [G]).

**Formula:**
The 2-D Convolution Theorem states:
$$ g_{x,y} * h_{x,y} = \sum_{a=0}^{N-1} \sum_{b=0}^{N-1} h_{a,b} \cdot g_{x-a,y-b} \Leftrightarrow G_{u,v} H_{u,v} $$
where $*$ is spatial convolution, $G$ and $H$ are frequency transforms, and $N$ is the dimension size assumed in the convolution sum limits [S2, unofficial].

**Parameters:**
- $N$: The dimension size of the input signal. The kernel must be padded to this size [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Circular Convolution:** FFT assumes periodic boundary conditions, which causes wrap-around artifacts if the kernel is large relative to the image or if the image has sharp edges [G].
- **Normalization:** Incorrect kernel normalization leads to energy gain or loss. The sum of all pixel positions in the kernel should equal 1 [S1, unofficial].
- **Precision:** Floating-point errors in FFT can accumulate [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frequency_domain_convolution(image, kernel):
    """
    Computes convolution using FFT.
    image: 2D array of shape (N_h, N_w)
    kernel: 2D array of shape (K_h, K_w)
    """
    N_h, N_w = len(image), len(image[0])
    K_h, K_w = len(kernel), len(kernel[0])
    N = max(N_h, N_w)
    
    # Zero-pad image and kernel to N x N as per source [S2, unofficial]
    G = [[complex(0, 0) for _ in range(N)] for _ in range(N)]
    H = [[complex(0, 0) for _ in range(N)] for _ in range(N)]
    for i in range(N_h):
        for j in range(N_w):
            G[i][j] = complex(image[i][j], 0)
    for i in range(K_h):
        for j in range(K_w):
            H[i][j] = complex(kernel[i][j], 0)
            
    # Forward FFT (standard 2D DFT algorithm) [G]
    G_freq = fft2d(G)
    H_freq = fft2d(H)
    
    # Frequency multiplication
    Y = [[G_freq[i][j] * H_freq[i][j] for j in range(N)] for i in range(N)]
    
    # Inverse FFT (standard 2D IDFT algorithm) [G]
    result = ifft2d(Y)
    
    # Extract real part
    return [[real(result[i][j]) for j in range(N)] for i in range(N)]
```

## Subtopics

### [Foveated Spatially Varying Lens Approximation](14591-foveated-spatially-varying-lens-approximation.md)
This method approximates spatially variant focus and aberration control by applying the correct lens phase function only at the user's tracked fovea position, rather than computing the full spatially variant solution for the entire hologram. It builds on [Frequency Domain Convolution](6341-frequency-domain-convolution.md) by substituting a fixed lens phase function with a temporally varying one derived from eye tracking. Unlike piece-wise methods, it provides a smooth and continuous falloff of aberration correction outside the foveal region without sharp phase discontinuities.

## Operating characteristics
- **Time Complexity:** Dominated by the FFT operations, making it computationally economical for large kernels [S2, unofficial].
- **Memory Complexity:** $O(N^2)$ to store the padded image and kernel in the frequency domain [G].
- **Strengths:** Efficient for large kernels [S2, unofficial].
- **Limitations:** Circular convolution artifacts require careful padding; high memory usage for large images [G].
- **When to use:** Large kernel blurring or optical system simulation where the kernel size is large [S1, unofficial][G].
- **When not to use:** Small kernels where spatial convolution is faster [G].

## Background
The method is based on the Convolution Theorem, which states that convolution in the spatial domain is equivalent to multiplication in the frequency domain [S2, unofficial]. It has been used in computer graphics and physics to simulate optical blurring and correct imperfections in optical systems by convolving with the inverse of a Point Spread Function (PSF) [S1, unofficial]. The discrete convolution may suffer from incorrect normalization of the filter, requiring the sum of all pixel positions in the kernel to equal 1 to conserve energy [S1, unofficial].

## Sources
[S1] Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
[S2] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
