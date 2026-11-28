# Frequency Domain Depth Cueing
## Overview
Frequency Domain Depth Cueing is a method to implement depth cueing (weighting voxel intensities by distance) in Fourier Volume Rendering (FVR) by convolving the 3-D Fourier transform with a combined filter $G(\mathbf{U})$ in the frequency domain, avoiding the need to recompute the 3-D forward transform for every view [S1, unofficial]. It sets itself apart from naive depth cueing by maintaining the computational advantage of FVR, which would otherwise be lost if the spatial dataset were depth-cued and re-transformed for each viewing angle.
## How it works
**Inputs and outputs:**
- Input: 3-D Fourier transform of the volume data, $X(\mathbf{U})$.
- Input: Reconstruction filter $H(\mathbf{U})$ used in standard FVR.
- Input: Depth cueing function parameters (e.g., decay rate).
- Output: Depth-cued 2-D projection image.

**Data structures:**
- $X(\mathbf{U})$: 3-D complex-valued array representing the Fourier transform of the volume $x(\mathbf{u})$.
- $H(\mathbf{U})$: 3-D filter kernel in frequency space.
- $G(\mathbf{U})$: Combined filter in frequency space that incorporates both reconstruction and depth cueing.

**Method:**
1. Define the spatial depth cueing function $d(\mathbf{u})$, which weights voxel intensities based on their distance from the viewer.
2. The depth-cued spatial dataset is $x(\mathbf{u}) \cdot d(\mathbf{u})$.
3. In the frequency domain, multiplication in space corresponds to convolution. The Fourier transform of the depth-cued data is $\mathcal{F}\{x(\mathbf{u}) \cdot d(\mathbf{u})\} = X(\mathbf{U}) * D(\mathbf{U})$, where $D(\mathbf{U})$ is the Fourier transform of the depth cueing function.
4. Standard FVR applies a reconstruction filter $H(\mathbf{U})$ via convolution: $X(\mathbf{U}) * H(\mathbf{U})$.
5. To combine depth cueing and reconstruction in the frequency domain, convolve the volume's Fourier transform with a combined filter $G(\mathbf{U})$.
6. The combined filter is defined as the convolution of the depth cueing filter and the reconstruction filter: $G(\mathbf{U}) = D(\mathbf{U}) * H(\mathbf{U})$.
7. The final rendered image is obtained by convolving $X(\mathbf{U})$ with $G(\mathbf{U})$:
   $$ \text{Image} = X(\mathbf{U}) * G(\mathbf{U}) $$
   This allows the depth cueing to be applied without recomputing the 3-D forward transform for every view [S1, unofficial].

**Parameters:**
- Depth cueing function $d(\mathbf{u})$: Typically an exponential decay function $e^{-\alpha \cdot \text{distance}}$.
- Reconstruction filter $H(\mathbf{U})$: Standard FVR filter (e.g., box, sinc).

**Edge cases and pitfalls:**
- Computing $G(\mathbf{U})$ requires convolving $D(\mathbf{U})$ and $H(\mathbf{U})$. If $D(\mathbf{U})$ is not precomputed or approximated, this step can be expensive.
- The method assumes the depth cueing function is separable or easily transformable.
- Precision issues may arise if $G(\mathbf{U})$ has significant sidelobes or if the convolution is not performed with sufficient resolution.

## Subtopics
## Operating characteristics
- **Time complexity:** Avoids $O(N^3 \log N)$ 3-D FFT per view. The main cost is the convolution $X(\mathbf{U}) * G(\mathbf{U})$, which can be optimized.
- **Memory complexity:** Requires storage for $X(\mathbf{U})$ and $G(\mathbf{U})$.
- **Strengths:** Maintains the speed advantage of FVR while supporting depth cueing.
- **Limitations:** Requires precomputation or efficient evaluation of $G(\mathbf{U})$. Not given in the sources for specific performance metrics.

## Background
Originated in the context of Fourier Volume Rendering (FVR) as a solution to the inefficiency of spatial depth cueing. The method is described in the HP Labs technical report "Fourier volume rendering" [S1, unofficial].

## Sources
[S1] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
