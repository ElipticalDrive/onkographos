# Text-guided Scene Style Editing

Also called: CLIP Loss for Text-Guided Style Editing, CLIP loss training

## Overview
Text-guided scene style editing uses CLIP semantic distance to control the appearance of a 3D scene via text prompts while maintaining multi-view consistency [S1, unofficial]. Unlike previous style editing methods that target single images, this method edits the entire scene, producing multi-view consistent images of the same style [S1, unofficial]. It builds on [CLIP Similarity Loss](14846-clip-similarity-loss.md) by introducing appearance embeddings and specific network architecture modifications to ProLiF [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Reference images of the scene, text prompts describing the desired style, and a pre-trained CLIP model.
*   **Outputs:** Rendered images with the edited style, maintaining the original scene structure.

**Data Structures:**
*   **Appearance Embeddings:** Feature codes concatenated to intermediate features of the ProLiF network. Two distinct appearance embeddings are used per training step: one for $L_{render}$ and one for $L_{CLIP}$ [S1, unofficial].
*   **ProLiF Network:** A progressively-connected light field network. The architecture is modified for style editing to include appearance embeddings [S1, unofficial].

**Parameters:**
*   $\lambda_{CLIP}$: Weight for the CLIP loss, set to 0.05 [S1, unofficial].
*   Pixel Sampling:
    *   16,384 pixels for $L_{render}$ [S1, unofficial].
    *   A $224 \times 224$ patch for $L_{CLIP}$ [S1, unofficial].
    *   4,096 extra pixels for $L_{reg}$ [S1, unofficial].

**Steps:**
1.  **Network Forward Pass:** Generate rendered images using the ProLiF network with appearance embeddings concatenated to intermediate features [S1, unofficial].
2.  **Loss Calculation:**
    *   Compute $L_{render}$ as the MSE loss between rendered pixels and reference image pixels using one appearance embedding [S1, unofficial].
    *   Compute $L_{CLIP}$ using a $224 \times 224$ patch and a second, different appearance embedding [S1, unofficial].
    *   Compute $L_{reg}$ using 4,096 extra pixels [S1, unofficial].
3.  **Stop-Gradient Application:** When calculating $L_{CLIP}$, apply stop-gradient operators to the output densities and a hidden feature of the appearance network [S1, unofficial]. This regularizes the rendered results to preserve the original intrinsic structure of the scene [S1, unofficial].
4.  **Total Loss:** Combine losses: $L = L_{render} + L_{reg} + \lambda_{CLIP} L_{CLIP}$ [S1, unofficial].

**Formulas:**
*   Total Loss:
    $$L = L_{render} + L_{reg} + \lambda_{CLIP} L_{CLIP}$$
*   CLIP Loss:
    $$L_{CLIP} = 1 - \frac{f_{image} \cdot f_{text}}{\|f_{image}\| \cdot \|f_{text}\|}$$
    Where $f_{image}$ and $f_{text}$ are the extracted features of the rendered image patch and the given text respectively by the CLIP model [S1, unofficial].

**Pitfalls:**
*   $L_{CLIP}$ and $L_{render}$ must be calculated from two different appearance embeddings to avoid conflicting optimization signals [S1, unofficial].
*   The method is built on a near-far-plane setting and cannot be used for 360° scenes [S1, unofficial].
*   Training is time-consuming compared to inference [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_style_editing_loss(rendered_image, reference_image, text_prompt, clip_model, appearance_embedding_1, appearance_embedding_2, lambda_clip=0.05):
    # Sample pixels for L_render
    render_pixels = sample_pixels(rendered_image, count=16384)
    ref_pixels = sample_pixels(reference_image, count=16384)
    l_render = mse_loss(render_pixels, ref_pixels)

    # Sample patch for L_clip
    clip_patch = extract_patch(rendered_image, size=(224, 224))
    
    # Get CLIP features
    f_image = clip_model.encode_image(clip_patch)
    f_text = clip_model.encode_text(text_prompt)
    
    # Compute CLIP loss
    cosine_sim = dot_product(f_image, f_text) / (norm(f_image) * norm(f_text))
    l_clip = 1 - cosine_sim

    # Sample pixels for L_reg
    reg_pixels = sample_pixels(rendered_image, count=4096, offset=16384)
    l_reg = compute_regularization_loss(reg_pixels) # Implementation details not specified in source [G]

    # Total loss
    total_loss = l_render + l_reg + lambda_clip * l_clip
    
    # Note: Stop-gradient operators are applied to output densities and hidden appearance features
    # when calculating L_clip in the actual backpropagation graph.
    return total_loss
```

## Operating characteristics
*   **Time Complexity:** Training is time-consuming, requiring network optimization for each scene [S1, unofficial]. Inference is fast [S1, unofficial].
*   **Memory Complexity:** Model size is 11.0MB for ProLiF, compared to 5.0MB for NeuLF [S1, unofficial].
*   **Performance:** Achieves faster rendering in inference stage [S1, unofficial]. Quantitative comparisons show better rendering quality than NeuLF on the Shiny Object dataset [S1, unofficial].
*   **Strengths:** Produces multi-view consistent images of the same style for the whole scene [S1, unofficial].
*   **Limitations:** Cannot be used for 360° scenes due to near-far-plane setting [S1, unofficial].

## Background
The method was introduced in the paper "Progressively-connected Light Field Network for Efficient View Synthesis" [S1, unofficial]. It extends the ProLiF architecture to support text-guided style editing using CLIP loss [S1, unofficial]. Previous style editing methods targeted single images, whereas this approach targets the entire 3D scene [S1, unofficial].

## Sources
[S1] Progressively-connected Light Field Network for Efficient View Synthesis, https://doi.org/10.48550/arxiv.2207.04465, fetched 2026-10-27 (unofficial)
