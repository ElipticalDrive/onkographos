# Text Embedding Generation with Directional Views


## Overview
This method generates semantic text embeddings conditioned on camera viewing directions to enable view-dependent semantic understanding in 3D Gaussian Splatting. It solves the problem of static semantic representations by appending directional descriptors (e.g., "front view") to text prompts before encoding, allowing the system to distinguish between different viewpoints of the same object. It differs from standard semantic encoding by explicitly coupling the text latent space with spatial orientation, whereas siblings like [LangSplat](8104-langsplat.md) may use more complex cross-attention mechanisms or frozen vision-language models without explicit directional token injection.

## How it works
**Inputs and Outputs:**
*   **Input:** A base text prompt string (e.g., "a chair"), a set of directional descriptors, and a frozen CLIP text encoder.
*   **Output:** A dictionary of tensors mapping directional keys to embedding vectors.

**Data Structures:**
*   `self.embeddings`: A dictionary where keys are strings (`'pos'`, `'neg'`, `'front'`, `'side'`, `'back'`) and values are tensors of shape `[1, 77, 768]` (batch size 1, sequence length 77, embedding dimension 768).

**Steps:**
1.  **Tokenization:** Tokenize the base prompt using the CLIP tokenizer with padding to the maximum model length.
    $$ \text{inputs} = \text{tokenizer}(\text{prompt}, \text{padding}=\text{"max\_length"}, \text{max\_length}=\text{model\_max\_length}, \text{return\_tensors}=\text{"pt"}) $$
2.  **Base Encoding:** Pass the token IDs through the frozen CLIP text encoder to obtain the base embedding.
    $$ \text{embeddings} = \text{text\_encoder}(\text{inputs.input\_ids})[0] $$
3.  **Directional Augmentation:** For each required direction $d \in \{\text{'front'}, \text{'side'}, \text{'back'}\}$, append the string `', {d} view'` to the original prompt.
4.  **Directional Encoding:** Tokenize and encode the augmented prompts using the same frozen text encoder. Store the resulting tensors in `self.embeddings` under the corresponding keys.
5.  **Storage:** Store the base positive/negative embeddings under keys `'pos'` and `'neg'` if required by the downstream segmentation or rendering pipeline.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_directional_embeddings(self, base_prompt, text_encoder, tokenizer, device):
    """
    Generates text embeddings for base prompt and specific directional views.
    """
    # Step 1 & 2: Encode base prompt
    inputs = tokenizer(
        base_prompt, 
        padding="max_length", 
        max_length=tokenizer.model_max_length, 
        return_tensors="pt"
    )
    base_embed = text_encoder(inputs.input_ids.to(device))[0]
    
    # Initialize storage dictionary
    self.embeddings = {
        'pos': base_embed,
        'neg': -base_embed # [G] Standard negation for contrastive loss
    }
    
    # Step 3 & 4: Encode directional views
    directions = ['front', 'side', 'back']
    for d in directions:
        # Append directional descriptor
        augmented_prompt = f"{base_prompt}, {d} view"
        
        # Tokenize and encode
        dir_inputs = tokenizer(
            augmented_prompt, 
            padding="max_length", 
            max_length=tokenizer.model_max_length, 
            return_tensors="pt"
        )
        dir_embed = text_encoder(dir_inputs.input_ids.to(device))[0]
        
        # Store in dictionary
        self.embeddings[d] = dir_embed
        
    return self.embeddings
```

**Parameters:**
*   **Direction Set:** `['front', 'side', 'back']` [S1, unofficial]. For forward-facing scenes (e.g., LLFF dataset), `'top-down view'` is excluded [S2].
*   **Sequence Length:** 77 tokens (standard CLIP limit) [S1, unofficial].
*   **Embedding Dimension:** 768 (CLIP ViT-L/14 text encoder output) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Prompt Syntax:** The comma and space before the direction descriptor (`, {d} view`) are critical for CLIP's tokenization consistency. Omitting them may result in invalid or semantically distinct tokens.
*   **Frozen Encoder:** The text encoder must remain frozen during the 3D Gaussian optimization process to maintain the stability of the semantic space [S1, unofficial].

## Subtopics

### Directional Dependent Prompts
A variant where the directional prompt suffix is not pre-computed but dynamically concatenated to the input caption at each training iteration based on the current camera location in the scene [S2]. This allows the model to adaptively condition the semantic representation on the instantaneous viewpoint rather than a fixed set of canonical views. It is chosen when dynamic viewpoint-dependent semantic consistency is required during optimization, such as in novel view synthesis with semantic constraints [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$, where $N$ is the number of directional views (typically 3-4) and $L$ is the sequence length (77). This is a one-time cost per prompt during initialization or prompt update.
*   **Memory Complexity:** $O(N \cdot L \cdot D)$, where $D$ is the embedding dimension (768). For 4 directions, this is negligible compared to Gaussian attributes.
*   **Performance:** Enables view-dependent semantic segmentation and editing. Metrics like CLIP Direction Similarity and CLIP Direction Consistency are used to evaluate the alignment between text changes and image changes [S2].
*   **Strengths:** Simple to implement; leverages robust pre-trained CLIP features; decouples semantic conditioning from geometry optimization.
*   **Limitations:** Limited to the discrete set of directions defined; does not support arbitrary continuous directional interpolation without additional interpolation steps.

## Background
Originating from extensions of 3D Gaussian Splatting for semantic and language-aware rendering, this method builds on the principle of differentiable feature accumulation. It was notably detailed in the context of TexGS-VolVis for text-driven non-photorealistic style editing [S1, unofficial] and Blended-NeRF for zero-shot object generation [S2]. The approach leverages the CLIP text encoder, which was introduced by Radford et al. (2021), to provide a shared latent space between images and text.

## Sources
[S1] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S2] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
