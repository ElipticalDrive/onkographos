# Text-Image Direction Score

Also called: TIDS

## Overview
The Text-Image Direction Score (TIDS) is a quantitative metric used in text-guided NeRF editing to estimate the semantic alignment between the direction of a text prompt change and the resulting change in the rendered image [S1]. Unlike pixel-level metrics such as MSE or PSNR, which measure visual fidelity, TIDS evaluates whether the edit performed by the model corresponds to the semantic intent of the instruction [S1]. It sets itself apart from sibling metrics like R-Precision by focusing on the directionality of the edit in embedding space rather than the retrieval accuracy of a single view against a caption [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $oI_i$: The original rendered image for view $i$.
    *   $eI_i$: The edited rendered image for view $i$.
    *   $oT_i$: The text caption/prompt describing the original scene.
    *   $eT_i$: The text caption/prompt describing the edited scene.
    *   $\mathbf{C}_I$: The CLIP image encoder [S1].
    *   $\mathbf{C}_T$: The CLIP text encoder [S1].
*   **Output:** A scalar value representing the cosine similarity between the image edit vector and the text edit vector.

**Data Structures:**
*   Embeddings are vectors of fixed dimensionality [G].

**Steps:**
1.  Encode the original rendered image $oI_i$ using the image encoder to obtain $\mathbf{C}_I(oI_i)$ [S1].
2.  Encode the edited rendered image $eI_i$ using the image encoder to obtain $\mathbf{C}_I(eI_i)$ [S1].
3.  Encode the original text prompt $oT_i$ using the text encoder to obtain $\mathbf{C}_T(oT_i)$ [S1].
4.  Encode the edited text prompt $eT_i$ using the text encoder to obtain $\mathbf{C}_T(eT_i)$ [S1].
5.  Compute the image direction vector: $\Delta \mathbf{V}_I = \mathbf{C}_I(eI_i) - \mathbf{C}_I(oI_i)$ [S1].
6.  Compute the text direction vector: $\Delta \mathbf{V}_T = \mathbf{C}_T(eT_i) - \mathbf{C}_T(oT_i)$ [S1].
7.  Calculate the cosine similarity between $\Delta \mathbf{V}_I$ and $\Delta \mathbf{V}_T$ [S1].

**Formula:**
$$ \text{TIDS} = \cos(\mathbf{C}_I(eI_i) - \mathbf{C}_I(oI_i), \mathbf{C}_T(eT_i) - \mathbf{C}_T(oT_i)) $$
$$ \text{TIDS} = \frac{(\mathbf{C}_I(eI_i) - \mathbf{C}_I(oI_i)) \cdot (\mathbf{C}_T(eT_i) - \mathbf{C}_T(oT_i))}{\|\mathbf{C}_I(eI_i) - \mathbf{C}_I(oI_i)\| \|\mathbf{C}_T(eT_i) - \mathbf{C}_T(oT_i)\|} $$
where $\cos$ denotes cosine similarity [S1].

**Parameters:**
*   **CLIP Model:** The standard CLIP model is used for both $\mathbf{C}_I$ and $\mathbf{C}_T$ [S1].
*   **Views:** The score is typically computed per view $i$ and averaged across the evaluation set [S1].

**Edge Cases and Pitfalls:**
*   **Zero Vectors:** If the image or text embeddings do not change (i.e., $\Delta \mathbf{V}_I = \mathbf{0}$ or $\Delta \mathbf{V}_T = \mathbf{0}$), the denominator becomes zero. Implementations must handle this by returning 0 or skipping the view, as the direction is undefined [G].
*   **Prompt Specificity:** The metric assumes that $oT_i$ and $eT_i$ accurately reflect the semantic difference intended by the user. Ambiguous prompts may yield low scores even if the visual edit is correct [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def text_image_direction_score(clip_image_encoder, clip_text_encoder, 
                               original_image, edited_image, 
                               original_text, edited_text):
    # 1. Encode images and text
    emb_o_img = clip_image_encoder(original_image)
    emb_e_img = clip_image_encoder(edited_image)
    emb_o_txt = clip_text_encoder(original_text)
    emb_e_txt = clip_text_encoder(edited_text)
    
    # 2. Compute direction vectors
    img_direction = emb_e_img - emb_o_img
    txt_direction = emb_e_txt - emb_o_txt
    
    # 3. Compute cosine similarity
    dot_product = torch.dot(img_direction.flatten(), txt_direction.flatten())
    img_norm = torch.norm(img_direction)
    txt_norm = torch.norm(txt_direction)
    
    # Handle zero vectors
    if img_norm == 0 or txt_norm == 0:
        return 0.0
    return dot_product / (img_norm * txt_norm)
```

## Subtopics

### [R-Precision](13785-r-precision.md)
R-Precision measures how well a rendered view of a synthesis object aligns with the text caption used to generate it. It computes the precision of rendered views over a group of text captions using a retrieval model. It is distinct from pixel-based metrics by evaluating semantic alignment rather than geometric fidelity. Choose it when evaluating text-to-3D generation quality rather than edit directionality. See [R-Precision](13785-r-precision.md).

## Operating characteristics
*   **Time Complexity:** Dominated by the inference time of the CLIP image and text encoders. For a single view, it is $O(1)$ relative to the number of pixels but depends on the encoder's depth [G].
*   **Memory Complexity:** Requires storing the CLIP model weights and the embeddings for the original and edited images/text [G].
*   **Performance:** In simple settings (e.g., 5 edits on NeRF-Art face scene), TIDS scores for methods like Instruct-NeRF2NeRF are around 0.1477, while ViCA-NeRF achieves 0.1545 [S1].
*   **Strengths:** Directly measures semantic alignment with the text instruction, which is the primary goal of text-guided editing [S1].
*   **Limitations:** Does not measure visual fidelity or photorealism [S1].

## Background
The Text-Image Direction Score was introduced in the context of evaluating text-guided NeRF editing methods [S1]. It was adopted from prior work on image editing evaluation to quantify how well the visual changes in a NeRF correspond to the textual instructions provided [S1]. It is used alongside consistency scores to evaluate methods like ViCA-NeRF and Instruct-NeRF2NeRF [S1].

## Sources
[S1] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
