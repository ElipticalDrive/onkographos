# Frequency Regularization

Also called: Amplitude and Phase Discrepancy Regularization, Geometry Regularization, Improved Optimization via Regularization

## Overview
Frequency Regularization (FR) is a training-time technique that minimizes the discrepancy between the frequency spectrum of rendered images and ground truth to alleviate over-reconstruction and improve Gaussian densification [S2]. It solves the problem where standard pixel-wise losses (e.g., L1) fail to differentiate between well-reconstructed and over-reconstructed regions, leading to redundant Gaussians and blurred details [S2]. Unlike naive spatial losses, FR explicitly disentangles frequency components, increasing pixel gradients in over-reconstructed regions to drive adaptive densification [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered image $\hat{I} \in \mathbb{R}^{H \times W \times C}$, Ground truth image $I \in \mathbb{R}^{H \times W \times C}$, Image dimensions $H, W$.
*   **Outputs:** Regularization loss $\mathcal{L}_{reg}$, gradients flowing back to Gaussian parameters $\theta$ (centers $\mu$, scales $s$, rotations $q$, opacities $\alpha$, colors $c$) [Parent, S2].

**Data Structures:**
*   Complex frequency arrays $F(u,v)$ and $\hat{F}(u,v)$ of size $H \times W$ per channel.
*   Amplitude arrays $|F|$ and $|\hat{F}|$ of size $H \times W$.
*   Phase arrays $\angle F$ and $\angle \hat{F}$ of size $H \times W$.

**Steps:**
1.  **Fourier Transform:** Compute the 2D Discrete Fourier Transform (DFT) for the ground truth image $I$ and rendered image $\hat{I}$ to obtain frequency representations $F(u,v)$ and $\hat{F}(u,v)$ [S2].
    $$ F(u, v) = \sum_{x=0}^{H-1} \sum_{y=0}^{W-1} I(x, y) \cdot e^{-i2\pi(\frac{u}{H}x + \frac{v}{W}y)} $$
    $$ \hat{F}(u, v) = \sum_{x=0}^{H-1} \sum_{y=0}^{W-1} \hat{I}(x, y) \cdot e^{-i2\pi(\frac{u}{H}x + \frac{v}{W}y)} $$
    where $(x, y)$ are spatial coordinates, $(u, v)$ are frequency coordinates [S2].

2.  **Amplitude Extraction:** Calculate the amplitude for each frequency component [S2]:
    $$ |F(u, v)| = \sqrt{\text{Re}(u, v)^2 + \text{Im}(u, v)^2} $$
    $$ |\hat{F}(u, v)| = \sqrt{\hat{\text{Re}}(u, v)^2 + \hat{\text{Im}}(u, v)^2} $$
    where $\text{Re}$ and $\text{Im}$ are the real and imaginary parts of the complex frequency value [S2].

3.  **Phase Extraction:** Calculate the phase for each frequency component [S2]:
    $$ \angle F(u, v) = \arctan\left(\frac{\text{Im}(u, v)}{\text{Re}(u, v)}\right) $$
    $$ \angle \hat{F}(u, v) = \arctan\left(\frac{\hat{\text{Im}}(u, v)}{\hat{\text{Re}}(u, v)}\right) $$
    [S2].

4.  **Discrepancy Calculation:** Compute the total amplitude discrepancy $d_a$ and total phase discrepancy $d_p$ using the Euclidean metric over all pixels [S2]:
    $$ d_a = \sqrt{\frac{1}{HW} \sum_{x=0}^{H-1} \sum_{y=0}^{W-1} \left( |F(u, v)| - |\hat{F}(u, v)| \right)^2} $$
    $$ d_p = \sqrt{\frac{1}{HW} \sum_{x=0}^{H-1} \sum_{y=0}^{W-1} \left( \angle F(u, v) - \angle \hat{F}(u, v) \right)^2} $$
    [S2].

5.  **Loss Computation:** The regularization loss is typically a weighted sum of these discrepancies (implementation detail: sources define $d_a$ and $d_p$ but do not explicitly specify the final scalar combination formula, though [S2] implies minimizing both). The gradients of $\mathcal{L}_{reg}$ are backpropagated through the differentiable rasterizer [Parent].

**Parameters:**
*   **Optimizer:** Adam [S2].
*   **Framework:** PyTorch [S2].
*   **Rasterization:** Custom CUDA kernels from 3D-GS [S2].
*   **Weight $\lambda_{reg}$:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Over-reconstruction Bias:** Naively applying amplitude and phase discrepancies uniformly across all frequencies can bias optimization towards undesirable artifacts and restrict Gaussian densification [S2]. This necessitates variants like Frequency Annealing [S2].
*   **Gradient Vanishing:** In well-reconstructed regions, spatial L1 loss yields small gradients, preventing densification. FR increases gradients in over-reconstructed regions, but precise tuning is required to avoid noise amplification [S2].

## Subtopics

### [Scale Regularization](9855-scale-regularization.md)
Scale regularization is a training-time technique that constrains the intrinsic scales of 4D Gaussians during deformation to prevent excessive dilation and improve anti-aliasing in video reconstruction. It solves the problem where standard Gaussian Splatting methods suffer from artifacts or loss of detail when tested at sampling rates different from training, by jointly regulating the maximum sampling frequency relative to the Gaussian's scale constraint. Unlike [Mip-Splatting](52-mip-splatting.md), which uses a fixed expansion scale, this method dynamically adapts frequency constraints, outperforming fixed-scale approaches in monocular video reconstruction.

### [Frequency Annealing](13483-frequency-annealing.md)
Frequency Annealing is a regularization technique for 3D Gaussian Splatting that mitigates over-reconstruction and artifacts by gradually shifting optimization focus from low-frequency (global structure) to high-frequency (local detail) components. It distinguishes itself from naive frequency regularization, which applies amplitude and phase discrepancies uniformly across all frequencies, by employing a dynamic high-pass filter that expands its spectral range over time to enable coarse-to-fine densification.

### DWTGS
DWTGS rethinks frequency regularization for sparse-view 3DGS by applying a discrete wavelet transform on the rendered image and penalizing the loss in the wavelet domain to encourage sharper edges [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The addition of 2D DFT and inverse operations per iteration increases computational overhead compared to pure spatial losses. The complexity is dominated by the FFT, $O(HW \log(HW))$ per channel [S2].
*   **Memory Complexity:** Requires storing intermediate frequency arrays $F$ and $\hat{F}$ for the current batch, increasing VRAM usage proportional to image resolution [S2].
*   **Performance:** FR significantly increases pixel gradients in over-reconstruction regions compared to L1 loss, enabling better adaptive densification [S2]. It alleviates over-reconstruction and improves Gaussian densification [S2].
*   **Strengths:** Explicitly addresses frequency-domain errors; effective in revealing over-reconstructed regions that spatial losses miss [S2].
*   **Limitations:** Naive application can cause artifacts; requires careful annealing or filtering to balance low and high frequencies [S2].
*   **When to use:** When over-reconstruction is prevalent or when spatial losses fail to drive densification in complex regions [S2].
*   **When not to use:** When computational resources are severely constrained due to FFT overhead, or if naive application leads to unstable artifacts without annealing [S2].

## Background
Frequency Regularization was introduced to address convergence speed, visual artifacts, and improper Gaussians in 3D Gaussian Splatting optimization [S1]. The specific method known as FreGS (3D Gaussian Splatting with Progressive Frequency Regularization) was proposed to boost 3DGS from a frequency perspective [S2]. It builds on the observation that pixel-wise L1 loss cannot differentiate over-reconstructed regions from well-reconstructed ones [S2]. The technique is part of a broader trend of adding regularization (e.g., geometry, frequency) to improve 3D GS optimization [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] FreGS: 3D Gaussian Splatting with Progressive Frequency Regularization, https://doi.org/10.1109/cvpr52733.2024.02024, fetched 2026-10-26
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S4] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
