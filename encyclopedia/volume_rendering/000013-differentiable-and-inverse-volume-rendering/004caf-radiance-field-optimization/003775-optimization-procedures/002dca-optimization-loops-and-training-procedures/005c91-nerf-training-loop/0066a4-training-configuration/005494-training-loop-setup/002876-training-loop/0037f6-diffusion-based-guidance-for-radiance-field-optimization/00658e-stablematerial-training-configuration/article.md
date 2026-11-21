# StableMaterial Training Configuration

Also called: StableMaterial finetuning, StableMaterial setup

## Overview
StableMaterial Training Configuration defines the hyperparameters and data pipeline for finetuning a 2D diffusion model prior to estimate physically plausible material properties (albedo and ORM) from input images [S1]. It adapts a pretrained Stable Diffusion UNet by freezing the backbone and training only a custom adapter and the UNet weights, reducing computational requirements compared to full model training [S1]. This configuration serves as the foundation for the StableMaterial model, which provides strong material priors for inverse rendering pipelines like MaterialFusion [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** RGB images resized to $256 \times 256$ resolution [S1].
*   **Input Conditioning:** A concatenation of a CLIP embedding and a pose representation [S1].
*   **Output:** Factorized 2D maps for albedo ($\hat{I}_d$) and ORM (Occlusion, Roughness, Metallicness) ($\hat{I}_{orm}$) [S1].

**Data Structures:**
*   **Pose Representation:** Follows the format defined in Zero123 [S1].
*   **Fully Connected (FC) Layer:** Converts the concatenated CLIP and pose embedding into a compatible embedding for the Stable Diffusion UNet. The layer maps from dimension 1028 to 1024 [S1].

**Training Steps:**
1.  **Data Filtering:** Filter the dataset (Objaverse) using inclusion keys `[pbr, pbrtexture, substance, substancepainter]` and exclusion keys `[style, stylized, cartoon, lowpoly, poly]` [S1]. Objects with procedural generation features are skipped [S1].
2.  **Preprocessing:** Resize input images to $256 \times 256$ resolution [S1].
3.  **Embedding Concatenation:** Concatenate the CLIP embedding and the pose representation [S1].
4.  **Adapter Projection:** Pass the concatenated embedding through the fully connected layer ($1028 \rightarrow 1024$) to produce the conditioning embedding for the UNet [S1].
5.  **Forward Pass:** Input the conditioning embedding and the image into the Stable Diffusion UNet [S1].
6.  **Loss Computation:** Compute the noise prediction loss (standard diffusion training objective) [S1].
7.  **Backpropagation:** Update the FC layer and the Stable Diffusion UNet weights. Freeze all other components of the Stable Diffusion model [S1].

**Parameters:**
*   **Batch Size:** 1536 [S1].
*   **Base Learning Rate:** $10^{-4}$ [S1].
*   **FC Layer Learning Rate:** $10^{-3}$ [S1].
*   **Optimizer:** AdamW [S1].
*   **Training Steps:** 25,000 [S1].
*   **Hardware:** 8× NVIDIA H100 GPUs [S1].
*   **Training Duration:** 2 days [S1].

**Pitfalls:**
*   Ensure input images are converted to RGB and sRGB tonemapped if required by the downstream pipeline (e.g., MaterialFusion's $L_{recon}$) [S1].
*   Do not train the UNet backbone weights from scratch; only finetune the adapter and UNet weights to reduce compute [S1].

## Subtopics

### Multi-view Attention at Inference
An inference-time modification to StableMaterial that enforces material consistency across multiple input views by modifying self-attention layers [S1]. It processes a batch of 4 images as a single large image within self-attention layers, allowing each latent pixel to attend to pixels in all other images, while non-self-attention layers process images independently [S1]. This mechanism is strictly for 2D inference and not used during 3D optimization or training [S1]. It improves albedo PSNR (24.70 vs 24.25) and handles difficult views better than the standard StableMaterial variant [S1].

## Operating characteristics
*   **Compute:** Requires 8× NVIDIA H100 GPUs for 2 days to complete 25,000 steps with a batch size of 1536 [S1].
*   **Performance:** The multi-view variant (StableMaterialMV) achieves an albedo PSNR of 24.70 and ORM L1 of 0.018, outperforming the single-view StableMaterial (PSNR 24.25) and baselines like IIR, IID, and Derender3D [S1].
*   **Limitations:** Multi-view attention is limited to 2D image inference and does not directly enhance 3D inverse rendering optimization, as the albedo/ORM representations are already optimized for multi-view consistency in the 3D pipeline [S1].
*   **Usage:** Best suited for generating material priors for inverse rendering (MaterialFusion) or for sparse-view 2D scenarios where 3D reconstruction is infeasible [S1].

## Background
StableMaterial was introduced in the MaterialFusion paper (2025) as a 2D diffusion model prior designed to refine multi-view data to estimate albedo and material properties (ORM) from input appearances [S1]. It addresses the challenge of recovering physically plausible material factors under unknown illumination by leveraging a fine-tuned Stable Diffusion architecture [S1]. The training configuration follows a similar approach to Zero123, adapting the Stable Diffusion UNet with a custom pose/CLIP adapter [S1].

## Sources
[S1] MaterialFusion: Enhancing Inverse Rendering with Material Diffusion Priors, https://doi.org/10.1109/3dv66043.2025.00079, fetched 2026-10-29
