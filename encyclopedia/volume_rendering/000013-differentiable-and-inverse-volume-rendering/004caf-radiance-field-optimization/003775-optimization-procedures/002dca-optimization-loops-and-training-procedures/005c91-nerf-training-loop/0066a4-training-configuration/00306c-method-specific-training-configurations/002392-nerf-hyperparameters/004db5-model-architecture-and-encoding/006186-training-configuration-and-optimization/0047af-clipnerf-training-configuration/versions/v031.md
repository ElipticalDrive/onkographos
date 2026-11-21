# CLIPNeRF Training Configuration

Also called: Absolute Directional CLIP Loss, CLIPNeRF Hyperparameters, CLIPNeRF Implementation, Directional CLIP Loss, Relative Directional CLIP Loss, StyleGAN-NADA CLIP Loss

## Overview
CLIPNeRF training configuration defines the optimization procedure for text-guided stylization of Neural Radiance Fields (NeRFs) by minimizing the negative cosine similarity between rendered image embeddings and target text prompt embeddings [S1]. It distinguishes itself from standard photometric NeRF training by replacing or augmenting pixel-wise color loss with semantic constraints in the CLIP embedding space, enabling style transfer without requiring paired 3D data [S2].

## How it works
**Inputs and Outputs:**
- **Input:** A pre-trained NeRF model (MLP parameters), a source image $I_{src}$ (optional, for relative loss), a target text prompt $t_{tgt}$, and a source text prompt $t_{src}$ (optional).
- **Output:** Updated NeRF parameters that stylize the rendered views to match the semantic direction of $t_{tgt}$ relative to $t_{src}$ or absolute style.

**Data Structures:**
- **NeRF Model:** Standard MLP mapping 3D coordinates and viewing directions to color and density. Parameters are split into density-related and color-related sub-parameters.
- **CLIP Encoder:** ViT-B/32 architecture [S1]. Takes an image or text and outputs a normalized embedding vector.
- **Patch Buffer:** Stores rendered pixel values for a $128 \times 128$ patch.

**Parameters:**
- **Iterations:** 500 [S1].
- **Optimizer:** Adam [S1].
- **Learning Rate:** $0.0001$ [S1].
- **Patch Size:** $128 \times 128$ pixels [S1].
- **Ray Sampling Stride:** 5 within the patch [S1]. This results in a receptive field of $640 \times 640$ on the original image scale [S1].
- **Importance Sampling Points:** 16 per ray during patch rendering [S1].
- **Fine MLP Total Samples:** 80 (64 coarse + 16 fine) [S1].

**Steps:**
1. **Freeze Density:** Freeze sub-parameters related to density in the NeRF to prevent structural changes during stylization [S1].
2. **Sample Ray:** Randomly sample a camera pose from the training dataset [S1].
3. **Render Patch:** Render a $128 \times 128$ patch from the sampled pose.
   - Rays are sampled with a stride of 5 to cover a wider area [S1].
   - Use importance sampling with 16 points per ray [S1].
   - The fine MLP uses 80 total samples (64 coarse, 16 fine) [S1].
4. **Resize:** Apply bilinear interpolation to resize the rendered patch to the input resolution required by the CLIP encoder [S1].
5. **Encode:** Pass the resized patch through the CLIP ViT-B/32 image encoder to get $\hat{e}_i$ [S1]. Encode the target text prompt $t_{tgt}$ to get $\hat{e}_t$ [S1].
6. **Compute Loss:** Calculate the loss based on the chosen directional strategy (see Formulas).
7. **Optimize:** Update the color-related NeRF parameters using Adam with learning rate $0.0001$ [S1].

**Formulas:**
Two variants of the directional CLIP loss are used:

1. **Absolute Directional CLIP Loss:**
   Measures cosine similarity between the rendered image embedding and the target text embedding [S2].
   $$ \mathcal{L}_{dir} = 1 - \langle \hat{e}_i(I_{tgt}), \hat{e}_t(t_{tgt}) \rangle $$
   Where $\langle \cdot, \cdot \rangle$ denotes the dot product of normalized vectors [S2]. This guides the NeRF with a global direction but ignores magnitude [S2].

2. **Relative Directional CLIP Loss:**
   Transfers style from a source image $I_{src}$ to the target domain using a pair of text prompts ($t_{src}, t_{tgt}$) [S2].
   $$ \mathcal{L}_{dir} = 1 - \langle \hat{e}_i(I_{tgt}) - \hat{e}_i(I_{src}), \hat{e}_t(t_{tgt}) - \hat{e}_t(t_{src}) \rangle $$
   This enforces the correct stylization trajectory relative to the source [S2].

**Edge Cases and Pitfalls:**
- **Memory Constraints:** Naive differentiable rendering of high-resolution images is intractable due to memory limits [S1]. The patch-based approach mitigates this [S1].
- **Mode Collapse:** Absolute directional CLIP loss can cause mode collapse of the generator and hurt generation diversity [S2].
- **Insufficient Magnitude:** Directional loss alone struggles to reach sufficient stylization magnitude, resulting in weak style transfer [S2]. This is because it measures directional similarity but ignores actual distances in embedding space [S2].
- **Non-Uniform Stylization:** Global contrastive loss may lead to excess stylization in some regions and insufficiency in others (e.g., eyes vs. faces) [S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of iterations (500) and the number of rays sampled per patch. Patch rendering is significantly faster than full-image rendering.
- **Memory Complexity:** Proportional to the patch size ($128 \times 128$) and sampling points (80 per ray), rather than full image resolution. This allows training on hardware with limited VRAM [S1].
- **Strengths:** Enables text-driven stylization without 3D ground truth; computationally efficient via patch rendering [S1].
- **Limitations:** May produce non-uniform stylization across the scene; absolute loss risks mode collapse; relative loss requires a source image and prompt [S2].
- **When to use:** When text-guided style transfer is required for NeRFs and computational resources are constrained [S1].
- **When not to use:** When high-fidelity, uniform stylization magnitude is critical without additional global contrastive mechanisms [S2].

## Background
CLIPNeRF was introduced by Wang et al. in 2021 [S2]. It extends the StyleCLIP [Patashnik et al. 2021] and StyleGAN-NADA [Gal et al. 2021] concepts to 3D neural radiance fields. The method was further refined in NeRF-Art [S2] to address issues with magnitude and uniformity. It is used for text-driven 3D scene stylization and editing [S2].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] NeRF-Art: Text-Driven Neural Radiance Fields Stylization, https://doi.org/10.1109/tvcg.2023.3283400, fetched 2026-10-26
