# Multi-pass GAN for Volumetric Super-Resolution
Also called: Multi-pass GAN (Werhahn et al.), Physics-informed Multi-pass GAN

## Overview
A physics-informed super-resolution method that decomposes volumetric upscaling into orthogonal 2D slice passes via a multi-pass GAN, incorporating governing equation constraints directly into the training objective to ensure physical consistency [S1, unofficial]. It differs from standard 3D GANs by avoiding the memory cost of full 3D convolutions, enabling high-resolution reconstruction (e.g., $512^3$) from low-resolution inputs ($64^3$) within seconds [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Low-resolution volumetric data (e.g., $64^3$ scalar fields representing smoke density).
- Output: High-resolution volumetric data (e.g., $512^3$ scalar fields).

**Data Structures:**
- The method utilizes two generator networks operating sequentially.
- Generator 1 ($G_1$): Processes 2D slices in the XY-plane.
- Generator 2 ($G_2$): Refines 2D slices in the YZ-plane.
- No explicit 3D convolution kernels are used; memory scales with 2D slice dimensions rather than the full 3D volume [S1, unofficial].

**Steps:**
1. **Orthogonal Decomposition:** The 3D volume is decomposed into a sequence of 2D slices along orthogonal axes (XY and YZ).
2. **First Pass (XY):** $G_1$ upscales the XY-plane slices from low resolution to high resolution. This step captures spatial details within the horizontal plane.
3. **Second Pass (YZ):** $G_2$ takes the output from $G_1$ and refines the YZ-plane slices. This step captures vertical structures and correlations that were not fully resolved in the first pass.
4. **Physics-Informed Training:** The training objective includes constraints derived from governing physical equations (e.g., Navier-Stokes or buoyancy equations for smoke) rather than relying solely on pixel-wise reconstruction error [S1, unofficial].
5. **Progressive Growing:** The network employs progressive growing with curriculum learning to stabilize training and enable large upscaling factors (8× per dimension) [S1, unofficial].

**Parameters:**
- Upscaling factor: 8× per dimension (e.g., $64^3 \rightarrow 512^3$) [S1, unofficial].
- Training time: Approximately 60 seconds for the described case [S1, unofficial].

**Edge Cases and Pitfalls:**
- The method was evaluated only on synthetic buoyant smoke data [S1, unofficial].
- No quantitative reconstruction metrics (such as PSNR or SSIM) were reported in the source [S1, unofficial].
- Performance on non-synthetic or non-smoke data is not documented [S1, unofficial].

## Subtopics

### STNet
An end-to-end GAN framework that extends super-resolution to joint spatial and temporal dimensions [S1, unofficial]. It performs feature-space temporal interpolation followed by voxel-shuffle spatial upscaling [S1, unofficial]. It uses a ConvLSTM-based discriminator to enforce spatiotemporal realism and includes a cycle-loss pre-training stage that utilizes unlabeled low-resolution data [S1, unofficial]. Unlike the multi-pass GAN which focuses on spatial physics, STNet optimizes PSNR-oriented losses [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Fast inference; produced $512^3$ volumes from $64^3$ inputs within 60 seconds [S1, unofficial].
- **Memory Complexity:** Reduced compared to 3D CNNs due to the decomposition into 2D slice passes, avoiding full 3D convolution memory costs [S1, unofficial].
- **Performance:** Competitive with direct numerical simulation (DNS) in terms of visual detail for turbulent structures [S1, unofficial].
- **Strengths:** Enables high-resolution ($512^3$) reconstruction at low computational cost; physics-informed constraints ensure physically plausible results.
- **Limitations:** Evaluated only on synthetic buoyant smoke; lack of quantitative metrics limits generalizability assessment; no reported performance on other data types [S1, unofficial].
- **When to use:** When high-resolution volumetric super-resolution is needed for physics-based simulations (e.g., smoke) and computational resources for 3D convolutions are limited.
- **When not to use:** When quantitative accuracy metrics are required or when dealing with non-synthetic/non-smoke data where physics constraints are not well-defined or validated.

## Background
Originated from work by Werhahn et al. [158] as cited in [S1, unofficial]. It represents an alternative line of work in scientific visualization that replaces standard CNNs with physics-informed GANs to achieve resolution-independent inference while maintaining physical fidelity. The approach addresses the high cost of direct numerical simulation by providing a fast, learned alternative that respects governing equations [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
