# Lanczos Resampling

Also called: Lanczos kernel

## Overview
Lanczos resampling is an interpolation technique used to resample (downsample or upsample) volumetric data by replacing each selected sample with a translated copy of the Lanczos kernel, which is a limited Sinc function [S1, unofficial]. It is chosen when higher fidelity than linear or cubic interpolation is required, particularly in medical imaging contexts where preserving high-frequency details during compression or reconstruction is critical [S1, unofficial]. It distinguishes itself from its siblings, such as [Interpolation Schemes for Volume Rendering](817-interpolation-schemes-for-volume-rendering.md) and [Interpolation Functions for Resampling](15932-interpolation-functions-for-resampling.md), by using a windowed sinc function that balances sharpness (low blurring) with the suppression of ringing artifacts compared to a pure sinc filter.

## How it works
**Inputs and Outputs:**
*   **Input:** A discrete set of volumetric samples $V_i$ located at grid positions $\mathbf{p}_i$.
*   **Query Position:** A 3D coordinate $\mathbf{p}$ along the ray $r(t) = \mathbf{o} + t\mathbf{d}$ where the value is needed.
*   **Output:** The interpolated scalar value $V(\mathbf{p})$.

**Data Structures:**
*   **Kernel Order ($a$):** An integer parameter defining the radius of the kernel in voxels. Typical values are $a=2$ or $a=3$ [S1, unofficial].
*   **Sample Grid:** The input volume texture or array.

**Formula:**
The Lanczos kernel $L(x)$ is defined as:
$$
L(x) = \begin{cases} 
\text{sinc}\left(\frac{\pi x}{a}\right) \text{sinc}(\pi x) & \text{if } |x| < a \\
0 & \text{otherwise}
\end{cases}
$$
Where:
*   $x$ is the distance from the query position $\mathbf{p}$ to the sample center $\mathbf{p}_i$ (i.e., $x = \|\mathbf{p} - \mathbf{p}_i\|$ in the relevant dimension).
*   $a$ is the Lanczos order (radius).
*   $\text{sinc}(u) = \frac{\sin(u)}{u}$. Note that $\text{sinc}(0) = 1$.

**Steps:**
1.  **Coordinate Transformation:** Map the world-space or camera-space query position $\mathbf{p}$ to the local coordinate system of the volume data (e.g., normalized $[-1, 1]$ or $[0, 1]$ ranges) to determine the continuous voxel coordinates [Parent].
2.  **Neighbor Selection:** Identify all sample indices $i$ such that the distance $|x|$ between the query position and the sample center $\mathbf{p}_i$ is less than the kernel order $a$. For a 3D volume, this typically involves a cubic neighborhood of size $(2a-1)^3$.
3.  **Weight Calculation:** For each selected neighbor $i$, calculate the distance $x$ along each axis (if separable) or the radial distance. Compute the weight $w_i = L(x)$ using the formula above.
4.  **Summation:** Compute the interpolated value:
    $$
    V(\mathbf{p}) = \sum_{i} w_i V_i
    $$
    [Parent]

**Pitfalls:**
*   **Ringing Artifacts:** Like all sinc-based filters, Lanczos resampling can introduce Gibbs phenomenon (ringing) near sharp edges if the kernel order is too high or if the signal is not band-limited [S1, unofficial].
*   **Boundary Conditions:** The kernel is zero outside $|x| < a$. Care must be taken at volume boundaries to avoid accessing out-of-bounds memory or applying incorrect weights.
*   **Precision:** The $\text{sinc}(0)$ case must be handled explicitly to avoid division by zero.

## Subtopics

### High-Quality Resampling (3D Lanczos via 2D Targets)
A technique to perform 3D Lanczos resampling on GPUs that lack native 3D render targets by decomposing the 3D filter into sequential 1D filters along the major axes, using 2D render targets and texture shifting [S2, unofficial]. This approach is necessary because native 3D render targets were not supported at the time of the source's writing [S2, unofficial].

**Steps (for Lanczos Order N=3):**
1.  **Z-Axis Pass:** Use the 3D input texture six times, shifted each time by one voxel along the z-axis. Use two weighting textures for the six weights. Sum the weighted input voxels and store the result in a $6 \times 6 \times 6$ intermediate output texture [S2, unofficial].
2.  **Y-Axis Pass:** Perform the same operation for the y-axis using the 6 previously generated intermediate textures, leading to six new intermediate output textures [S2, unofficial].
3.  **X-Axis Pass:** Use these textures, each shifted by one pixel along the x-axis, leading to the final texture [S2, unofficial].

**Parameters:**
*   **Lanczos Order N:** Determines the number of passes and texture shifts. For $N=3$, 43 passes are required ($36+6+1$). For $N=2$, 21 passes are required [S2, unofficial].

**Pitfalls:**
*   **Hardware Limitations:** The split-axis approach is a workaround for GPU limitations regarding 3D render targets. It requires significant memory bandwidth and multiple rendering passes, which can be computationally expensive [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(a^3)$ per sample, where $a$ is the kernel order. The high-quality GPU variant requires $O(N^3)$ passes for order $N$ if implemented naively, but the split-axis method reduces this to a fixed number of passes per axis (e.g., 43 passes for $N=3$) [S2, unofficial].
*   **Memory Complexity:** $O(1)$ for standard CPU/GPU texture sampling. The split-axis GPU method requires storing intermediate textures, increasing memory usage proportional to the number of passes and texture dimensions [S2, unofficial].
*   **Performance:** Provides higher PSNR (Peak Signal-to-Noise Ratio) compared to lower-order interpolations, making it suitable for high-resolution medical image compression and reconstruction [S1, unofficial]. However, it is computationally more expensive than linear or cubic interpolation.
*   **Strengths:** Sharp reconstruction with minimal blurring; good balance between aliasing and smoothing.
*   **Limitations:** Susceptible to ringing artifacts; computationally intensive for high orders or large volumes.

## Background
Lanczos resampling is named after the mathematician Cornelius Lanczos. It is widely used in signal processing and image resampling. In the context of volume rendering, it has been applied to compress high-resolution medical images using Implicit Neural Representations (INRs) like SIREN, where it serves as a preprocessing step to downsample HR images before neural network training [S1, unofficial]. It has also been used in Fourier volume rendering on the GPU to implement windowed-sinc functions for frequency space extraction [S2, unofficial].

## Sources
[S1] A Novel Implicit Neural Representation for Volume Data., https://eprints.bournemouth.ac.uk/38355/, fetched 2026-10-29 (unofficial)
[S2] Fourier volume rendering on the GPU using a split- ..., https://www.researchgate.net/publication/220838994_Fourier_volume_rendering_on_the_GPU_using_a_split-stream-FFT, fetched 2026-10-31 (unofficial)
