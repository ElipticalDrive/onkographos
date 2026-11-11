# Multi-pass GAN for Volumetric Super-Resolution

Also called: Multi-pass GAN (Werhahn et al.), Physics-informed Multi-pass GAN

## Overview
This method performs spatial super-resolution on volumetric data by decomposing the 3D reconstruction problem into orthogonal 2D slice passes, thereby avoiding the memory costs associated with full 3D convolutions [S1, unofficial]. It distinguishes itself from standard 3D GANs by integrating physics-informed constraints directly into the training objective to ensure physically consistent reconstructions, rather than relying solely on statistical error minimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Low-resolution volumetric data (e.g., $64^3$ scalar fields representing density or velocity).
*   **Output:** High-resolution volumetric data (e.g., $512^3$).

**Data Structures:**
*   **Generators:** Two distinct generator networks, $G_{XY}$ and $G_{YZ}$.
    *   $G_{XY}$: Processes 2D slices in the XY-plane.
    *   $G_{YZ}$: Processes 2D slices in the YZ-plane.
*   **Discriminator:** Standard GAN discriminator structure (architecture details not specified in sources) [S1, unofficial].

**Method Steps:**
1.  **Decomposition:** The volumetric super-resolution task is decomposed into orthogonal 2D passes. The method does not use 3D convolutions [S1, unofficial].
2.  **First Pass ($G_{XY}$):** The first generator upscales the input volume by processing it as a sequence of XY-plane slices.
3.  **Second Pass ($G_{YZ}$):** The second generator refines the output from the first pass by processing it as a sequence of YZ-plane slices.
4.  **Physics-Informed Training:** The training objective incorporates constraints derived from governing equations (e.g., Navier-Stokes equations for fluid flow) directly into the loss function. This ensures the reconstructed field adheres to physical laws [S1, unofficial].
5.  **Progressive Growing:** The network is trained using progressive growing with curriculum learning to handle the high resolution ($512^3$) [S1, unofficial].

**Parameters:**
*   **Upscaling Factor:** 8× per dimension (e.g., $64^3 \rightarrow 512^3$) [S1, unofficial].
*   **Training Time:** Approximately 60 seconds for the reported $64^3 \rightarrow 512^3$ case [S1, unofficial].

**Pitfalls:**
*   **Evaluation Limitations:** The method was evaluated only on synthetic buoyant smoke data [S1, unofficial].
*   **Metrics:** No quantitative reconstruction metrics (such as PSNR or SSIM) were reported in the source [S1, unofficial].
*   **Generalizability:** Performance on non-synthetic or non-smoke data is not documented [S1, unofficial].

## Subtopics

### STNet
STNet is an end-to-end GAN framework that extends super-resolution to joint spatial and temporal dimensions [S1, unofficial]. It performs feature-space temporal interpolation followed by voxel-shuffle spatial upscaling [S1, unofficial]. It utilizes a ConvLSTM-based discriminator to enforce spatiotemporal realism and includes a cycle-loss pre-training stage that leverages unlabeled low-resolution data [S1, unofficial]. Unlike methods focusing solely on spatial resolution, STNet optimizes PSNR-oriented losses to maintain fidelity across time [S1, unofficial].

## Operating characteristics
*   **Memory Efficiency:** By decomposing 3D SR into 2D slice passes, the method avoids the memory cost of full 3D convolutions [S1, unofficial].
*   **Performance:** Capable of producing detailed turbulent structures at $512^3$ resolution from $64^3$ inputs within 60 seconds [S1, unofficial].
*   **Accuracy:** Results are described as competitive with direct numerical simulation (DNS) [S1, unofficial].
*   **Limitations:** Evaluated exclusively on synthetic buoyant smoke; lacks quantitative metrics [S1, unofficial].

## Background
The method was proposed by Werhahn et al. [S1, unofficial]. It belongs to the broader category of physics-informed super-resolution, which seeks physically consistent reconstructions by incorporating governing equation constraints into the training objective [S1, unofficial]. This approach contrasts with purely statistical reconstruction error minimization [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
