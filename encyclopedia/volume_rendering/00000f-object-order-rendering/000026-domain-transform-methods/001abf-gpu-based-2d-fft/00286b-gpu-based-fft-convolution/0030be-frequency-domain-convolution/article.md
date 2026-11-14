# Frequency Domain Convolution
Also called: FFT Convolution, Point Spread Function (PSF) Convolution

## Overview
Frequency Domain Convolution computes the convolution of an image or volume slice with a kernel by transforming both into the frequency domain, multiplying their transforms, and inverse-transforming the result. It is computationally economical for large kernels where the $O(N^2 \log N)$ cost of the Fast Fourier Transform (FFT) is lower than the $O(N^2 K^2)$ cost of spatial convolution [S2, unofficial]. Unlike direct spatial methods, it leverages the Convolution Theorem to replace summation with element-wise multiplication [S2, unofficial].

## How it works
**Inputs:**
- Input signal $g$ of size $N \times N$ (e.g., an image or volume slice).
- Convolution kernel $h$ of size $K_h \times K_w$ (typically $K_h, K_w \ll N$).

**Outputs:**
- Convolved signal $g * h$ of size $N \times N$.

**Data Structures:**
- Complex-valued arrays of size $N \times N$ for frequency domain representation.
- Zero-padded kernel array of size $N \times N$.

**Steps:**
1. **Zero-Pad Kernel:** Create an array $H_{pad}$ of size $N \times N$. Place the kernel $h$ at the top-left corner (or centered, depending on FFT convention, but standard implementation often places at $(0,0)$ and handles shift via phase or assumes circular convolution alignment). Fill the rest with zeros.
2. **Forward FFT:** Compute the 2D Discrete Fourier Transform (DFT) of the input signal $g$ to get $G_{u,v}$. Compute the 2D DFT of the padded kernel $H_{pad}$ to get $H_{u,v}$.
   $$ G_{u,v} = \text{FFT2}(g_{x,y}) $$
   $$ H_{u,v} = \text{FFT2}(H_{pad}) $$
3. **Frequency Multiplication:** Multiply the transforms element-wise.
   $$ Y_{u,v} = G_{u,v} \cdot H_{u,v} $$
4. **Inverse FFT:** Compute the 2D Inverse DFT of $Y_{u,v}$ to obtain the result in the spatial domain.
   $$ g'_{x,y} = \text{IFFT2}(Y_{u,v}) $$
5. **Normalization:** Ensure the kernel sum is 1 to conserve energy. If $\sum h \neq 1$, scale the result or the kernel before FFT [S1, unofficial].

**Formula:**
The 2-D Convolution Theorem states:
$$ g_{x,y} * h_{x,y} = \sum_{a=0}^{N-1} \sum_{b=0}^{N-1} h_{a,b} \cdot g_{x-a,y-b} \Leftrightarrow G_{u,v} H_{u,v} $$
where $*$ is spatial convolution, $G$ and $H$ are frequency transforms, and $N$ is the dimension size assumed in the convolution sum limits [S2, unofficial].

**Parameters:**
- $N$: The dimension size of the input signal. The kernel must be padded to this size [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Circular Convolution:** FFT assumes periodic boundary conditions. This causes "wrap-around" artifacts if the kernel is large relative to the image or if the image has sharp edges. Zero-padding the input signal to size $(N+K_h-1) \times (N+K_w-1)$ before FFT prevents this [S2, unofficial].
- **Normalization:** Incorrect kernel normalization leads to energy gain or loss. The sum of all pixel positions in the kernel should equal 1 [S1, unofficial].
- **Precision:** Floating-point errors in FFT can accumulate. Use double precision if high accuracy is required.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frequency_domain_convolution(image, kernel):
    """
    Computes convolution using FFT.
    image: 2D numpy array of shape (N, N)
    kernel: 2D numpy array of shape (K_h, K_w)
    """
    N_h, N_w = image.shape
    K_h, K_w = kernel.shape
    
    # Determine output size to avoid circular convolution artifacts
    # Standard practice is to pad to at least N + K - 1
    out_h = N_h + K_h - 1
    out_w = N_w + K_w - 1
    
    # Zero-pad image
    image_padded = np.zeros((out_h, out_w), dtype=np.complex128)
    image_padded[:N_h, :N_w] = image.astype(np.complex128)
    
    # Zero-pad kernel
    kernel_padded = np.zeros((out_h, out_w), dtype=np.complex128)
    kernel_padded[:K_h, :K_w] = kernel.astype(np.complex128)
    
    # Forward FFT
    G = np.fft.fft2(image_padded)
    H = np.fft.fft2(kernel_padded)
    
    # Element-wise multiplication
    Y = G * H
    
    # Inverse FFT
    result = np.fft.ifft2(Y)
    
    # Take real part and crop to original size
    result = np.real(result[:N_h, :N_w])
    
    # Normalize if necessary (assuming kernel sum != 1)
    kernel_sum = np.sum(kernel)
    if kernel_sum != 0:
        result = result / kernel_sum
        
    return result
```

## Subtopics

### [Foveated Spatially Varying Lens Approximation](14591-foveated-spatially-varying-lens-approximation.md)
This method approximates spatially variant focus and aberration control by applying the correct lens phase function only at the user's tracked fovea position, rather than computing the full spatially variant solution for the entire hologram. It builds on [Frequency Domain Convolution](6341-frequency-domain-convolution.md) by substituting a fixed lens phase function with a temporally varying one derived from eye tracking. Unlike piece-wise methods, it provides a smooth and continuous falloff of aberration correction outside the foveal region without sharp phase discontinuities.

## Operating characteristics
- **Time Complexity:** $O(N^2 \log N)$ for the FFTs, where $N$ is the image dimension. This is significantly faster than spatial convolution $O(N^2 K^2)$ for large kernels $K$ [S2, unofficial].
- **Memory Complexity:** $O(N^2)$ to store the padded image and kernel in the frequency domain.
- **Strengths:** Efficient for large kernels; leverages highly optimized FFT libraries (e.g., CUDA FFT) [S2, unofficial].
- **Limitations:** High memory usage for large images; circular convolution artifacts require careful padding; not suitable for real-time small-kernel operations where spatial convolution is faster.
- **When to use:** Large kernel blurring, optical system simulation, or when the kernel size is comparable to the image size [S2, unofficial].
- **When not to use:** Small kernels (e.g., $3 \times 3$), real-time applications with strict latency constraints, or when boundary effects cannot be managed via padding.

## Background
The method is based on the Convolution Theorem, which states that convolution in the spatial domain is equivalent to multiplication in the frequency domain [S2, unofficial]. It has been used in computer graphics and physics to simulate optical blurring and correct imperfections in optical systems by convolving with the inverse of a Point Spread Function (PSF) [S1, unofficial]. The use of FFT for convolution in volume rendering and image processing was highlighted in early works on Fourier volume rendering [S2, unofficial].

## Sources
[S1] Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
[S2] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
