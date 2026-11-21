# SSRTVD

Also called: SSR-TVD, tempoGAN

## Overview
SSRTVD is the first Generative Adversarial Network (GAN) designed for spatial super-resolution of time-varying volumetric data, specifically targeting fluid flow density fields [S1, unofficial]. It solves the problem of frame-to-frame inconsistency that arises when spatial super-resolution is applied independently to each timestep, which distorts features in motion [S1, unofficial]. It sets itself apart from sibling methods by introducing a temporal discriminator that operates on triplets of consecutive volumes to enforce coherence across timesteps, a mechanism absent in single-volume GANs like standard SR-GANs [S1, unofficial]. Unlike [TSR-TVD](10414-tsr-tvd.md), which focuses on temporal interpolation, SSRTVD focuses on spatial upscaling while maintaining temporal smoothness via advection-aligned constraints [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of coarse volumetric density fields $\{V_t\}_{t=1}^T$ representing time-varying fluid flow (e.g., smoke simulation).
*   **Output:** A sequence of high-resolution volumetric density fields $\{\hat{V}_t\}_{t=1}^T$ upscaled by a factor of 4 in each spatial dimension (64× volumetric increase) [S1, unofficial].

**Data Structures:**
*   **Volumetric Grid:** 3D tensor of shape $(D, H, W)$ for each timestep. The source specifies resolutions up to $256^3$ [S1, unofficial].
*   **Advection-Aligned Frames:** The temporal discriminator requires frames aligned according to the fluid velocity field to account for motion between timesteps [S1, unofficial].

**Network Architecture:**
1.  **Generator ($G$):** A densely connected convolutional network that maps coarse volumes to high-resolution volumes [S1, unofficial].
2.  **Spatial Discriminator ($D_{spatial}$):** A standard discriminator that evaluates the realism of individual high-resolution volumes $\hat{V}_t$ [S1, unofficial].
3.  **Temporal Discriminator ($D_{temporal}$):** A discriminator that processes triplets of consecutive volumes $(\hat{V}_{t-1}, \hat{V}_t, \hat{V}_{t+1})$ to enforce temporal coherence [S1, unofficial].

**Training Steps:**
1.  **Advection Alignment:** For each triplet of consecutive coarse volumes $(V_{t-1}, V_t, V_{t+1})$, apply a differentiable semi-Lagrangian advection layer to align $V_{t-1}$ and $V_{t+1}$ with $V_t$ based on the velocity field [S1, unofficial]. This creates advection-aligned frames.
2.  **Generator Forward Pass:** Pass the aligned coarse frames through $G$ to produce high-resolution predictions $\hat{V}_{t-1}, \hat{V}_t, \hat{V}_{t+1}$ [S1, unofficial].
3.  **Spatial Discrimination:** Compute the spatial adversarial loss by comparing $\hat{V}_t$ against real high-resolution ground truth $V_t^{HR}$ using $D_{spatial}$ [S1, unofficial].
4.  **Temporal Discrimination:** Pass the triplet $(\hat{V}_{t-1}, \hat{V}_t, \hat{V}_{t+1})$ through $D_{temporal}$ [S1, unofficial].
5.  **Gradient Propagation:** The temporal discriminator passes gradients back through the differentiable semi-Lagrangian advection layer, allowing the generator to learn temporal smoothness constraints directly [S1, unofficial].
6.  **Loss Calculation:** The total loss combines the spatial adversarial loss, the temporal adversarial loss, and likely a reconstruction loss (e.g., MSE), though the specific weights are not detailed in the sources [S1, unofficial].
7.  **Optimization:** Update $G$, $D_{spatial}$, and $D_{temporal}$ using standard GAN optimization algorithms (e.g., Adam) [S1, unofficial].

**Parameters:**
*   **Upscaling Factor:** 4× spatial (64× volumetric) [S1, unofficial].
*   **Resolution:** Tested up to $256^3$ [S1, unofficial].
*   **Dataset:** Computer graphics smoke simulation data [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Per-Dataset Training:** The method requires training a separate model for each dataset, limiting generalization [S1, unofficial].
*   **Inference Time:** Inference can take up to eight hours for large volumes due to sub-volume GPU processing constraints [S1, unofficial].
*   **Evaluation Limitations:** Original evaluation was qualitative; no PSNR or SSIM was reported for the initial tempoGAN formulation, though later citations cite PSNR ranges for SSR-TVD [S1, unofficial].

## Subtopics

### TSR-TVD
TSR-TVD is a recurrent generative network that adapts the temporal discriminator concept to temporal interpolation rather than spatial super-resolution [S1, unofficial]. It synthesizes intermediate timesteps in time-varying volumetric data using a generator that combines ConvLSTM for temporal modeling with a voxel shuffle layer for feature-space upscaling [S1, unofficial]. It makes both forward and backward predictions, blended with learned weights [S1, unofficial]. It could only synthesize intermediate volumes at integer time positions [S1, unofficial]. It achieved 23–27 dB PSNR across combustion, supernova, and vortex datasets, outperforming linear interpolation on both PSNR and isosurface similarity [S1, unofficial].

### Self-Attention GAN for Volume SR
This variant refines the GAN paradigm by incorporating self-attention into the generator encoder to capture global spatial dependencies beyond the local convolutional receptive field [S1, unofficial]. It achieved improved perceptual scores at scale factors up to 8× on ionization and hurricane data [S1, unofficial]. However, the evaluation reports SSIM improvement without quantifying absolute PSNR relative to SSR-TVD on matched splits, and the maximum tested volume ($240 \times 240 \times 48$) is smaller than prior GAN baselines [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Inference is computationally expensive, taking up to eight hours for large volumes ($256^3$) due to sub-volume GPU processing [S1, unofficial].
*   **Memory Complexity:** Requires sufficient GPU memory to process sub-volumes; specific memory footprint is not given, but the need for sub-volume processing implies constraints on total volume size per batch [S1, unofficial].
*   **Performance:** At 4× upscaling, SSR-TVD achieved 34–41 dB PSNR across combustion, ionization, and vortex datasets [S1, unofficial]. It outperformed interpolation baselines on both PSNR and isosurface similarity [S1, unofficial].
*   **Strengths:** Enforces temporal coherence in spatial super-resolution, recovering fine-grained structures absent from coarse simulations [S1, unofficial].
*   **Limitations:** Requires per-dataset training; original evaluation lacked quantitative metrics (PSNR/SSIM) for the base tempoGAN formulation; inference is slow [S1, unofficial].
*   **When to Use:** When spatial super-resolution of time-varying fluid data is required and temporal consistency is critical, and per-dataset training is acceptable.
*   **When Not to Use:** When real-time inference is required, or when a generalizable model across multiple datasets is needed without retraining.

## Background
SSRTVD (also known as tempoGAN) was introduced by Xie et al. as the first GAN-based spatial super-resolution method for time-varying data [S1, unofficial]. It was originally developed for computer graphics smoke simulation [S1, unofficial]. The method established the temporal discriminator as a mechanism for enforcing coherence across timesteps by using advection-aligned frames and a differentiable semi-Lagrangian advection layer [S1, unofficial]. It is cited in the context of scientific volume rendering and deep learning methods for scientific data [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
