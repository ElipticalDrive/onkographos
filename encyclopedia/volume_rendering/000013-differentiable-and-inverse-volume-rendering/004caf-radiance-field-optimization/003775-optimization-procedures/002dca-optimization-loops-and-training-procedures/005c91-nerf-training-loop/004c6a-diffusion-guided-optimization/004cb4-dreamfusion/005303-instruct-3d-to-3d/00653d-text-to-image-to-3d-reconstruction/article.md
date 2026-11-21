# Text-to-Image-to-3D Reconstruction

Also called: Two-step text-to-3D

## Overview
Text-to-Image-to-3D Reconstruction is a two-step procedure that reconstructs a 3D object from a text prompt by first generating a 2D image using a diffusion model, then lifting that image to 3D using RealFusion [S1]. It differs from one-step text-to-3D methods [33] by offering easier user control over the generated appearance and potentially addressing diversity issues in generation [S1].

## How it works
**Inputs:** A text prompt $P$.
**Outputs:** A 3D radiance field (NeRF) representing the object.

**Data Structures:**
1.  **2D Diffusion Model:** A pretrained text-to-image model (e.g., Stable Diffusion) capable of sampling images from noise conditioned on $P$ [S1].
2.  **RealFusion:** A single-image 3D reconstruction method that optimizes a NeRF to match a single input image and its depth/normal estimates [S1].

**Steps:**
1.  **Image Generation:** Sample a large number of 2D images from the diffusion model conditioned on the text prompt $P$ [S1].
2.  **Image Selection:** The user selects a single desired image $I_{sel}$ from the generated samples [S1].
3.  **3D Reconstruction:** Lift $I_{sel}$ to 3D using RealFusion [S1].
    *   RealFusion optimizes a NeRF $\mathcal{N}$ to minimize the difference between rendered views of $\mathcal{N}$ and the input image $I_{sel}$ [S1].
    *   The optimization uses an L2 reconstruction loss [S1].
    *   View-specific prompting (e.g., "back view") can be used during the RealFusion optimization to mitigate the Janus problem [S1].

**Parameters:**
*   **Diffusion Model:** Not specified in sources [S1].
*   **RealFusion Loss:** L2 loss is used; LPIPS and SSIM were tested but excluded for simplicity/computational cost [S1].

**Edge Cases and Pitfalls:**
*   **Janus Problem:** Reconstructions may exhibit two or more faces because the loss function attempts to make every view resemble the input image [S1]. View-specific prompting alleviates but does not eliminate this [S1].
*   **Floaters:** Artifacts may appear; image-specific prompts help alleviate this [S1].
*   **Diversity:** One-step methods may suffer from lack of diversity; this two-step approach allows user selection to address this [S1].

## Subtopics

### Dream Fields
Dream Fields combines NeRF with CLIP to generate diverse 3D objects solely from natural language descriptions by optimizing the radiance field via multi-view constraints based on CLIP scores on the image caption [S2, unofficial]. It differs from Text-to-Image-to-3D by using CLIP scores directly as a loss signal rather than generating an intermediate image [S2, unofficial]. Choose this method when direct text-to-3D optimization without intermediate image selection is preferred [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Easier user control compared to one-step text-to-3D [S1]. May address diversity of generation issues [S1]. Single-image textual inversion is usually not necessary as sampled images are well-aligned with prompts [S1].
*   **Limitations:** Susceptible to the Janus problem [S1].
*   **When to use:** When user control over the specific appearance of the generated 3D object is required [S1].

## Background
The method is described in the context of RealFusion, which explores reconstructing 3D objects from text prompts via an intermediate image [S1]. It is compared to one-step text-to-3D approaches [33] [S1]. Dream Fields, a related concept, was introduced by [JMB* 21] [S2, unofficial].

## Sources
[S1] RealFusion 360° Reconstruction of Any Object from a Single Image, https://doi.org/10.1109/cvpr52729.2023.00816, fetched 2026-10-30
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
