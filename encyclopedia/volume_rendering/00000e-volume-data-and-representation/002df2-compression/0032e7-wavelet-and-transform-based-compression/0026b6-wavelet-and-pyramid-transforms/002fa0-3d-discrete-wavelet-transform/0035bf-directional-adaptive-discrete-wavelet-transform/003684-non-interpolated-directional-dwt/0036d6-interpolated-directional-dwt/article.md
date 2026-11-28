# Interpolated Directional DWT
Also called: iDA-DWT
## Overview
Interpolated Directional DWT is a variant of the Directional Adaptive Discrete Wavelet Transform (DA-DWT) that permits direction vectors with fractional coordinates ($d \in \mathbb{R}^3$) rather than restricting them to integer grid steps [S1]. It solves the problem of applying wavelet prediction and update operations along arbitrary directions by sub-sampling and interpolating values on the grid $\Pi$ [S1]. It differs from its parent, Non-interpolated Directional DWT, which uses integer direction vectors to ensure causality without sub-sampling [S1, 7756-non-interpolated-directional-dwt.md]. While it allows for arbitrary directionality, it introduces interpolation complexity and generally yields negligible or slightly negative compression gains compared to integer-based adaptive methods [S1].

## How it works
**Inputs and Outputs:**
*   Input: A volumetric data grid $\Pi$ and a set of direction vectors $d \in \mathbb{R}^3$ for each spatial location [S1].
*   Output: Wavelet coefficients (approximation and detail subbands) [G].

**Data Structures:**
*   The grid is conceptually divided into two sets of samples, $\Pi_0$ and $\Pi_1$, corresponding to the even and odd positions relative to the decomposition lattice, ensuring causality constraints are met during the inverse transform [S1].

**Interpolation Filter:**
To compute prediction and update values at non-integer coordinates, a 1D Lanczos interpolation filter $L(x)$ is applied [S1]. The filter is defined as:

$$
L(x) = \begin{cases} 
\text{sinc}(x) \cdot \text{sinc}(x/a) & \text{if } -a < x < a \\
0 & \text{otherwise}
\end{cases}
$$

Where:
*   $\text{sinc}(x) = \frac{\sin(x)}{x}$
*   $a$ is the Lanczos interpolation parameter, set to $a=2$ [S1].

**Algorithm Steps:**
1.  **Direction Selection:** For each voxel, determine the optimal direction vector $d \in \mathbb{R}^3$, allowing fractional components [S1].
2.  **Causality-Constrained Prediction:**
    *   Identify the target sample location for the prediction step based on $d$ [G].
    *   Retrieve neighboring samples from the grid subset $\Pi_0$ only [S1].
    *   Compute the interpolated value using the filter $L(x)$ [S1].
    *   Subtract this interpolated value from the current sample to generate the detail coefficient [G].
3.  **Causality-Constrained Update:**
    *   Identify the target sample location for the update step based on $d$ [G].
    *   Retrieve neighboring samples from the grid subset $\Pi_1$ only [S1].
    *   Compute the interpolated value using the filter $L(x)$ [S1].
    *   Add this interpolated value to the approximation coefficient [S1].
4.  **Sub-sampling:** Apply standard sub-sampling to separate the approximation and detail coefficients [G].

**Pitfalls and Edge Cases:**
*   **Causality Violation:** The inverse transform requires strict causality. The interpolation function must use *only* samples from $\Pi_0$ during the predict step and *only* samples from $\Pi_1$ during the update step. Violating this makes the transform non-invertible without side information [S1].
*   **Precision:** Fractional coordinates require floating-point arithmetic for interpolation, increasing computational cost compared to integer-only variants [S1].

## Subtopics

### Interpolated DA-DWT
Interpolated DA-DWT (iDA-DWT) is the specific application of interpolated direction vectors within the Directional Adaptive DWT framework [S1]. It enables the use of arbitrary direction vectors in prediction and update steps [S1]. However, empirical results show it does not improve compression efficiency and increases algorithmic complexity [S1]. For 32×32×32 code-blocks and bit-rates between 0.50 and 3.0 bpp using an in-slice (6,6) kernel, the average Bjøntegaard Delta PSNR (BD-PSNR) and BD-Bitrate (BD-BR) are:
*   iDA-(4,4,0): 0.023 dB / -0.215%
*   iDA-(4,4,2): 0.008 dB / -0.087% [S1]
It may have a slightly negative impact on image quality for some datasets [S1].

## Operating characteristics
*   **Complexity:** Higher computational complexity than Non-interpolated Directional DWT due to the need for floating-point interpolation and sub-sampling [S1].
*   **Performance:** Negligible or slightly negative gains in compression efficiency (BD-PSNR/BD-BR) compared to integer-based DA-DWT variants [S1].
*   **Limitations:** Increased algorithmic complexity without corresponding quality benefits; potential for slight quality degradation [S1].
*   **When to use:** Rarely recommended for general volumetric compression due to poor cost-benefit ratio; primarily of theoretical interest or for specific feature-aligned scenarios where integer directions are insufficient [S1].

## Background
The Interpolated Directional DWT was introduced as an extension to the Directional Adaptive DWT to allow direction vectors with fractional coordinates ($d \in \mathbb{R}^3$) [S1]. It builds on the Lanczos interpolation method described in [32] and extends the DA-DWT framework analyzed in [33] [S1]. The method was investigated in the context of volumetric medical image compression, specifically for JPEG 2000 extensions [S1].

## Sources
[S1] Wavelet based volumetric medical image compression, https://doi.org/10.1016/j.image.2014.12.007, fetched 2026-10-26
