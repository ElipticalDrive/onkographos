# Frequency Domain Convolution

Also called: FFT Convolution, Point Spread Function (PSF) Convolution

## Overview
Frequency Domain Convolution computes the spatial convolution of an image or volume slice by transforming the input and kernel into the frequency domain, performing element-wise multiplication, and inverse-transforming the result. It is computationally economical for large kernels where the cost of the Fast Fourier Transform (FFT) is outweighed by the reduction in arithmetic operations compared to direct spatial convolution [S2, unofficial].

## How it works
**Inputs:**
- Input signal $g$ (e.g., an image or volume slice) of size $N \times N$.
- Convolution kernel $h$ (e.g., a Point Spread Function) of size $M \times M$.

**Outputs:**
- Convolved signal $g * h$ of size $N \times N$ (assuming valid convolution or appropriate padding).

**Data Structures:**
- Complex-valued arrays for frequency domain representation.
- Zero-padded arrays to handle circular convolution artifacts.

**Steps:**
1. **Padding:** Pad the kernel $h$ with zeros to match the dimensions of the input $g$ ($N \times N$). This is necessary because the FFT implicitly assumes periodic boundary conditions, and zero-padding prevents spatial aliasing (wrap-around effects) [S2, unofficial].
2. **Forward Transform:** Compute the 2D Discrete Fourier Transform (DFT) of the padded kernel to obtain $H_{u,v}$ and the input $g$ to obtain $G_{u,v}$ [S2, unofficial].
3. **Multiplication:** Perform element-wise complex multiplication in the frequency domain: $Y_{u,v} = G_{u,v} \cdot H_{u,v}$ [S2, unofficial].
4. **Inverse Transform:** Compute the 2D Inverse Discrete Fourier Transform (IDFT) of $Y_{u,v}$ to obtain the spatial result [S2, unofficial].
5. **Normalization:** Ensure the kernel sums to 1 to conserve energy, preventing artificial brightening or darkening of the output [S1, unofficial].

**Formula:**
The 2-D Convolution Theorem states that spatial convolution is equivalent to frequency multiplication:
$$ g_{x,y} * h_{x,y} = \sum_{a=0}^{N-1} \sum_{b=0}^{N-1} h_{a,b} \cdot g_{x-a,y-b} \Leftrightarrow G_{u,v} H_{u,v} $$
where $*$ denotes spatial convolution, $G$ and $H$ are the frequency transforms, and $N$ is the dimension size assumed in the convolution sum limits [S2, unofficial].

**Parameters:**
- $N$: The dimension size of the input signal. The kernel must be padded to this size [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Normalization:** If the sum of the kernel values is not 1, the output energy will be scaled incorrectly [S1, unofficial].
- **Boundary Conditions:** Without proper zero-padding, circular convolution artifacts will appear at the image boundaries [S2, unofficial].
- **Kernel Size:** For small kernels, direct spatial convolution is often faster due to FFT overhead. Frequency domain convolution becomes advantageous only when the kernel size is sufficiently large [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N^2 \log N)$ for both forward and inverse FFTs, compared to $O(N^2 M^2)$ for direct spatial convolution.
- **Memory Complexity:** $O(N^2)$ for storing the complex frequency domain arrays.
- **Strengths:** Efficient for large kernels; leverages highly optimized GPU FFT libraries (e.g., cuFFT) [S2, unofficial].
- **Limitations:** High overhead for small kernels; requires padding which increases memory usage; assumes periodic boundaries unless padded [S2, unofficial].
- **When to use:** When the kernel size is large relative to the image, or when the same kernel is applied to many images (pre-computing the kernel's FFT).
- **When not to use:** For small kernels (e.g., $3 \times 3$) where direct convolution is faster.

## Background
The method is based on the Convolution Theorem, which relates convolution in the spatial domain to multiplication in the frequency domain. It is widely used in computer graphics for blurring images and correcting optical imperfections via Point Spread Function (PSF) convolution [S1, unofficial]. The approach is computationally economical for large kernels, as the speed of frequency domain multiplication balances the cost of the inverse transform [S2, unofficial].

## Sources
[S1] Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
[S2] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
