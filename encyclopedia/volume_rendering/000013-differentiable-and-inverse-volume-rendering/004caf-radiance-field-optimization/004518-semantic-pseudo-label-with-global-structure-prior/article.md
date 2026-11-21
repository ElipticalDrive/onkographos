# Semantic Pseudo Label with Global Structure Prior

Also called: DINO-ViT CLS Token Loss

## Overview
This method enforces semantic consistency between a reference view and unseen rendered views in radiance field optimization by using a pre-trained Vision Transformer (ViT) as a global structure prior [S1]. Unlike methods that rely on pixel-wise alignment or projected image embeddings, it extracts a single [CLS] token from DINO-ViT to represent the entire image, making the supervision robust to pixel-wise misalignment between views [S1]. It builds on [Radiance-field optimization](44-radiance-field-optimization.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** A reference view image $I_{ref}$, a rendered view image $I_{unseen}$ (patch $B$), and the DINO-ViT model.
*   **Outputs:** A scalar loss value $\mathcal{L}_{\text{cls}}$ used to update the radiance field parameters.

**Data Structures:**
*   **DINO-ViT Model:** A self-supervised Vision Transformer trained on ImageNet [S1]. It outputs a sequence of tokens for each input image.
*   **CLS Token:** The first token in the ViT output sequence, serving as a global representation of the input image [S1].

**Steps:**
1.  **Feature Extraction:** Pass the reference view patch $A$ and the unseen view patch $B$ through the pre-trained DINO-ViT model.
2.  **Token Selection:** Extract the [CLS] token from the output of the ViT for both patches. Let $f_{\text{vit}}(A)$ be the [CLS] token for patch $A$ and $f_{\text{vit}}(B)$ be the [CLS] token for patch $B$ [S1].
3.  **Loss Calculation:** Compute the L2 distance (squared Euclidean distance) between the two extracted [CLS] tokens [S1].

**Formula:**
The semantic loss is defined as:
$$
\mathcal{L}_{\text{cls}} = ||f_{\text{vit}}(A) - f_{\text{vit}}(B) ||^2
$$
where $f_{\text{vit}}(\cdot)$ denotes the extraction of the [CLS] token from the DINO-ViT output, and $A$ and $B$ are patches from the reference view and an unseen view, respectively [S1].

**Parameters:**
*   **ViT Model:** DINO-ViT [S1].
*   **Training Data for ViT:** ImageNet [S1].

**Edge Cases and Pitfalls:**
*   **Misalignment:** The method is designed to be robust to pixel-wise misalignment between views because the [CLS] token captures global semantic content rather than local pixel details [S1].
*   **Patch Selection:** $A$ and $B$ are patches. The specific size or selection criteria for these patches are not detailed in the provided sources, though they correspond to regions in the reference and unseen views [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_semantic_loss(reference_patch, unseen_patch, dino_vit_model):
    """
    Computes the semantic pseudo-label loss using DINO-ViT CLS tokens.
    
    Args:
        reference_patch (Tensor): Image patch from the reference view.
        unseen_patch (Tensor): Image patch from the rendered unseen view.
        dino_vit_model (nn.Module): Pre-trained DINO-ViT model.
        
    Returns:
        Tensor: Scalar loss value.
    """
    # Extract features using DINO-ViT
    # The model returns a sequence of tokens; we take the first one (CLS token)
    ref_features = dino_vit_model(reference_patch)
    unseen_features = dino_vit_model(unseen_patch)
    
    # Extract the [CLS] token (assumed to be the first token in the sequence)
    # [G] Assuming standard ViT output shape [batch, seq_len, dim], 
    # we select index 0 for the CLS token.
    cls_ref = ref_features[:, 0, :]
    cls_unseen = unseen_features[:, 0, :]
    
    # Compute L2 distance squared
    # [G] Using torch.norm or direct subtraction depending on framework, 
    # here we use explicit squared L2 norm.
    diff = cls_ref - cls_unseen
    loss = torch.sum(diff ** 2)
    
    return loss
```

## Subtopics

### Feature Correspondence via DiNO-ViT
This variant uses DiNO-ViT features to establish semantic correspondences between source and target objects, typically for appearance transfer or alignment [S2]. It differs from the global structure prior by operating on pixel-level features and cosine similarity rather than global [CLS] tokens and L2 distance [S2]. It is chosen when fine-grained semantic matching between specific parts of objects is required [S2].
*   **Definition:** Uses DiNO-ViT features to establish semantic correspondences between source and target objects [S2].
*   **Steps:**
    1.  Randomly render 100 images per object [S2].
    2.  Sample 5,000 non-background pixels from each image to compute features and affinities [S2].
    3.  For each sampled pixel, compute cosine similarity to all feature descriptors in the source NeRF [S2].
    4.  If objects are not pre-aligned, optimize rotation and translation of the target object to roughly align it with the source object [S2].
*   **Formula:** Cosine similarity between feature vectors $f_1$ and $f_2$:
    $$
    \operatorname{sim}(f_1, f_2) = \frac{\langle f_1, f_2 \rangle}{||f_1|| \cdot ||f_2||}
    $$
*   **Pitfalls:** Performance decreases on un-textured images because DiNO was trained on natural images [S2]. Edge loss (washed-out details) can occur due to 2D-to-3D mapping noise [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Robust to pixel-wise misalignment; provides semantic-level supervision [S1].
*   **Limitations:** Relies on the quality of the pre-trained ViT; may not capture fine-grained geometric details [S1].
*   **When to use:** When semantic consistency is more important than pixel-perfect alignment, or when dealing with views that have significant misalignment [S1].
*   **When not to use:** When fine-grained geometric accuracy is required without additional geometric priors [S1].

## Background
The method was introduced in SinNeRF [S1]. It leverages the property that Vision Transformers (ViT) capture semantic appearance after self-supervised pre-training, making them robust to misalignment [S1]. It builds on the observation that deep networks can learn invariant representations of content and style [S1].

## Sources
[S1] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
[S2] NeRF Analogies: Example-Based Visual Attribute Transfer for NeRFs, https://doi.org/10.1109/cvpr52733.2024.00444, fetched 2026-10-26
