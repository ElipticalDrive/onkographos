# SSRTVD
Also called: SSR-TVD, tempoGAN

## Overview
SSRTVD is the first GAN-based spatial super-resolution method for time-varying volumetric data, designed to resolve frame-to-frame inconsistency that arises when spatial SR is applied independently to each timestep [S1, unofficial]. It distinguishes itself from single-volume GANs by employing a temporal discriminator on triplets of consecutive volumes to enforce temporal coherence, and differs from temporal interpolation methods like TSR-TVD by focusing on spatial upscaling with advection-aligned constraints [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of coarse volumetric density fields $\{V_t\}_{t=1}^T$ representing time-varying fluid flow [S1, unofficial].
*   **Output:** A sequence of high-resolution volumetric density fields $\{\hat{V}_t\}_{t=1}^T$ upscaled by a factor of 4 in each spatial dimension (64× volumetric increase) [S1, unofficial].

**Data Structures:**
*   **Volumetric Grid:** 3D tensor of shape $(D, H, W)$ for each timestep, tested up to $256^3$ [S1, unofficial].
*   **Advection-Aligned Frames:** Frames aligned according to the fluid velocity field to account for motion between timesteps [S1, unofficial].

**Network Architecture:**
1.  **Generator ($G$):** A densely connected convolutional network mapping coarse volumes to high-resolution volumes [S1, unofficial].
2.  **Spatial Discriminator ($D_{spatial}$):** Evaluates the realism of individual high-resolution volumes $\hat{V}_t$ [S1, unofficial].
3.  **Temporal Discriminator ($D_{temporal}$):** Processes triplets of consecutive volumes $(\hat{V}_{t-1}, \hat{V}_t, \hat{V}_{t+1})$ to enforce temporal coherence [S1, unofficial].

**Training Steps:**
1.  **Advection Alignment:** Apply a differentiable semi-Lagrangian advection layer to align consecutive coarse volumes $(V_{t-1}, V_t, V_{t+1})$ with $V_t$ based on the velocity field, producing advection-aligned frames [S1, unofficial].
2.  **Generator Forward Pass:** Pass the aligned coarse frames through $G$ to produce high-resolution predictions $\hat{V}_{t-1}, \hat{V}_t, \hat{V}_{t+1}$ [S1, unofficial].
3.  **Spatial Discrimination:** Compute the spatial adversarial loss by comparing $\hat{V}_t$ against real high-resolution ground truth using $D_{spatial}$ [S1, unofficial].
4.  **Temporal Discrimination:** Pass the triplet $(\hat{V}_{t-1}, \hat{V}_t, \hat{V}_{t+1})$ through $D_{temporal}$ [S1, unofficial].
5.  **Gradient Propagation:** Pass gradients from $D_{temporal}$ back through the differentiable semi-Lagrangian advection layer, allowing $G$ to learn temporal smoothness constraints directly [S1, unofficial].
6.  **Loss & Optimization:** The exact loss formulation weights and optimization algorithm are not given in the sources [S1, unofficial].

**Parameters:**
*   **Upscaling Factor:** 4× spatial (64× volumetric) [S1, unofficial].
*   **Resolution:** Tested up to $256^3$ [S1, unofficial].
*   **Dataset:** Computer graphics smoke simulation data [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Per-Dataset Training:** Requires training a separate model for each dataset, limiting generalization [S1, unofficial].
*   **Inference Time:** Inference takes up to eight hours for large volumes due to sub-volume GPU processing constraints [S1, unofficial].
*   **Evaluation Limitations:** Initial evaluation was qualitative with no PSNR or SSIM reported [S1, unofficial].

## Subtopics

### TSR-TVD
A recurrent generative network that adapts the temporal discriminator concept to temporal interpolation rather than spatial super-resolution [S1, unofficial]. It synthesizes intermediate timesteps using a generator combining ConvLSTM for temporal modeling with a voxel shuffle layer for feature-space upscaling, making forward and backward predictions blended with learned weights [S1, unofficial]. It could only synthesize intermediate volumes at integer time positions [S1, unofficial]. It achieved 23–27 dB PSNR across combustion, supernova, and vortex datasets, outperforming linear interpolation on both PSNR and isosurface similarity [S1, unofficial].

### Self-Attention GAN for Volume SR
Refines the GAN paradigm by incorporating self-attention into the generator encoder to capture global spatial dependencies beyond the local convolutional receptive field [S1, unofficial]. It achieved improved perceptual scores at scale factors up to 8× on ionization and hurricane data [S1, unofficial]. The evaluation reports SSIM improvement without quantifying absolute PSNR relative to SSR-TVD on matched splits, and the maximum tested volume ($240 \times 240 \times 48$) is smaller than prior GAN baselines [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Inference is computationally expensive, taking up to eight hours for large volumes ($256^3$) due to sub-volume GPU processing [S1, unofficial].
*   **Memory Complexity:** Requires sufficient GPU memory for sub-volume processing; specific memory footprint is not given [S1, unofficial].
*   **Performance:** At 4× upscaling, achieved 34–41 dB PSNR across combustion, ionization, and vortex datasets [S1, unofficial]. Outperformed interpolation baselines on both PSNR and isosurface similarity [S1, unofficial].
*   **Strengths:** Enforces temporal coherence in spatial SR, recovering fine-grained structures absent from coarse simulations [S1, unofficial].
*   **Limitations:** Requires per-dataset training; initial evaluation lacked quantitative metrics; inference is slow [S1, unofficial].
*   **When to Use:** Spatial SR of time-varying fluid data where temporal consistency is critical and per-dataset training is acceptable [S1, unofficial].
*   **When Not to Use:** Real-time inference is required, or generalization across multiple datasets without retraining is needed [S1, unofficial].

## Background
Introduced by Xie et al. as the first GAN-based spatial SR method for time-varying data [S1, unofficial]. Originally developed for computer graphics smoke simulation [S1, unofficial]. Established the temporal discriminator as a mechanism for enforcing coherence across timesteps using advection-aligned frames and a differentiable semi-Lagrangian advection layer [S1, unofficial]. Cited in the context of scientific volume rendering and deep learning methods for scientific data [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
