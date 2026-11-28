# Multi-pass GAN for Volumetric Super-Resolution

Also called: Multi-pass GAN (Werhahn et al.), Physics-informed Multi-pass GAN

## Overview
This method performs spatial super-resolution on volumetric data by decomposing the 3D reconstruction problem into orthogonal 2D slice passes, thereby avoiding the memory costs associated with full 3D convolutions [S1, unofficial]. It distinguishes itself from standard 3D GANs by integrating physics-informed constraints directly into the training objective to ensure physically consistent reconstructions, rather than relying solely on statistical error minimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Low-resolution volumetric data, typically scalar fields such as density or velocity at $64^3$ resolution [S1, unofficial].
*   **Output:** High-resolution volumetric data at $512^3$ resolution [S1, unofficial].

**Data Structures:**
*   **Generators:** Two distinct generator networks, $G_{XY}$ and $G_{YZ}$ [S1, unofficial].
    *   $G_{XY}$: Processes 2D slices in the XY-plane [S1, unofficial].
    *   $G_{YZ}$: Processes 2D slices in the YZ-plane [S1, unofficial].
*   **Discriminator:** Architecture details are not specified in the sources [S1, unofficial].

**Method Steps:**
1.  **Decomposition:** The volumetric super-resolution task is decomposed into orthogonal 2D passes [S1, unofficial].
2.  **First Pass ($G_{XY}$):** The first generator upscales the input volume by processing it as a sequence of XY-plane slices [S1, unofficial].
3.  **Second Pass ($G_{YZ}$):** The second generator refines the output from the first pass by processing it as a sequence of YZ-plane slices [S1, unofficial].
4.  **Physics-Informed Training:** The training objective incorporates constraints derived from governing equations directly into the loss function [S1, unofficial].
5.  **Progressive Growing:** The network is trained using progressive growing with curriculum learning to handle the high resolution [S1, unofficial].

**Formulas:** Not given in the sources.

**Parameters:**
*   **Upscaling Factor:** 8× per dimension (e.g., $64^3 \rightarrow 512^3$) [S1, unofficial].
*   **Training Time:** Approximately 60 seconds for the reported $64^3 \rightarrow 512^3$ case [S1, unofficial].

**Pitfalls:**
*   **Evaluation Limitations:** The method was evaluated only on synthetic buoyant smoke data [S1, unofficial].
*   **Metrics:** No quantitative reconstruction metrics (such as PSNR or SSIM) were reported in the source [S1, unofficial].
*   **Generalizability:** Performance on non-synthetic or non-smoke data is not documented [S1, unofficial].

## Subtopics

### [SSRTVD](10431-ssrtvd.md)
SSRTVD is a GAN designed for spatial super-resolution of time-varying volumetric data, specifically targeting fluid flow density fields. It sets itself apart from single-volume GANs by introducing a temporal discriminator that operates on triplets of consecutive volumes to enforce frame-to-frame coherence, which prevents motion distortion. Choose it when spatial upscaling of fluid flows requires temporal smoothness without altering timestep resolution. [SSRTVD](10431-ssrtvd.md)

### STNet
STNet is an end-to-end GAN framework that extends super-resolution to joint spatial and temporal dimensions [S1, unofficial]. It sets itself apart by performing feature-space temporal interpolation followed by voxel-shuffle spatial upscaling, optimizing PSNR-oriented losses rather than purely spatial metrics [S1, unofficial]. Choose it when both spatial resolution and temporal consistency must be improved simultaneously [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** Memory usage is reduced by decomposing 3D SR into 2D slice passes, avoiding the cost of full 3D convolutions [S1, unofficial]. Training time is approximately 60 seconds for the reported $64^3 \rightarrow 512^3$ case [S1, unofficial].
*   **Measured Performance:** Produces detailed turbulent structures at $512^3$ resolution from $64^3$ inputs on synthetic buoyant smoke [S1, unofficial].
*   **Scaling:** Handles 8× upscaling per dimension via progressive growing and curriculum learning [S1, unofficial].
*   **Strengths and Limitations:** Competitive with direct numerical simulation (DNS) in visual fidelity, but lacks quantitative metrics and is untested on non-synthetic data [S1, unofficial].
*   **Usage Guidance:** Use for physics-consistent spatial upscaling of synthetic fluid flows where memory constraints prohibit 3D convolutions; avoid for production pipelines requiring validated quantitative error bounds or non-smoke data [S1, unofficial].

## Background
The method was proposed by Werhahn et al. [S1, unofficial]. It belongs to the broader category of physics-informed super-resolution, which seeks physically consistent reconstructions by incorporating governing equation constraints into the training objective [S1, unofficial]. This approach contrasts with purely statistical reconstruction error minimization [S1, unofficial]. It is primarily used in scientific visualization for smoke and fluid flow simulations [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
