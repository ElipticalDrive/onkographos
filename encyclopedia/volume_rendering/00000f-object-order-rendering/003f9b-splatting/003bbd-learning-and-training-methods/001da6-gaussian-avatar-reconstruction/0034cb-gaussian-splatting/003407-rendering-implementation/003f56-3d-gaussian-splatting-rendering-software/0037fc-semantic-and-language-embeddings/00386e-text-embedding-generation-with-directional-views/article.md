# Text Embedding Generation with Directional Views


## Overview
This method generates text embeddings for 3D Gaussian Splatting by encoding language prompts into latent vectors using a frozen CLIP text encoder [S1, unofficial]. It distinguishes itself from standard text embedding by appending directional view descriptors (e.g., "front", "side") to the prompt, allowing the system to associate specific semantic concepts with specific camera viewpoints [S1, unofficial][S2]. This enables directional-dependent rendering and open-vocabulary queries that respect spatial orientation [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A base text prompt string (e.g., "a red car") and a set of directional view identifiers.
*   **Output:** A dictionary of tensors mapping keys (`'pos'`, `'neg'`, `'front'`, `'side'`, `'back'`) to embedding tensors of shape `[1, 77, 768]` (batch size 1, sequence length 77, embedding dimension 768) [S1, unofficial].

**Data Structures:**
*   `self.embeddings`: A dictionary storing the pre-computed embeddings.
*   `self.tokenizer`: The CLIP tokenizer instance.
*   `self.text_encoder`: The frozen CLIP text encoder instance.

**Steps:**
1.  **Base Encoding:** Tokenize the base prompt with padding to the maximum sequence length (`model_max_length`, typically 77 for CLIP) [S1, unofficial].
2.  **Encode Base:** Pass the token IDs through the frozen text encoder to obtain the base embedding [S1, unofficial].
3.  **Directional Encoding:** For each directional view $d$ in the set $\{'front', 'side', 'back'\}$:
    a. Construct the directional prompt by appending `', {d} view'` to the base prompt [S1, unofficial].
    b. Tokenize and encode this new prompt.
    c. Store the result in `self.embeddings` under the key $d$ [S1, unofficial].
4.  **Storage:** Store all resulting tensors in `self.embeddings` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_directional_embeddings(self, base_prompt: str):
    """
    Generates text embeddings for a base prompt and its directional variants.
    """
    # 1. Tokenize base prompt
    inputs = self.tokenizer(
        base_prompt,
        padding="max_length",
        max_length=self.tokenizer.model_max_length,
        return_tensors="pt"
    )
    
    # 2. Encode base prompt
    # [S1, unofficial] Note: inputs.input_ids must be moved to the correct device
    base_embeddings = self.text_encoder(inputs.input_ids.to(self.device))[0]
    
    # Store base embeddings under 'pos' (positive) key
    self.embeddings['pos'] = base_embeddings
    
    # Define directional views
    directions = ['front', 'side', 'back']
    
    # 3. Encode directional variants
    for d in directions:
        # Construct directional prompt
        directional_prompt = f"{base_prompt}, {d} view"
        
        # Tokenize
        dir_inputs = self.tokenizer(
            directional_prompt,
            padding="max_length",
            max_length=self.tokenizer.model_max_length,
            return_tensors="pt"
        )
        
        # Encode
        dir_embeddings = self.text_encoder(dir_inputs.input_ids.to(self.device))[0]
        
        # Store
        self.embeddings[d] = dir_embeddings

    # Note: 'neg' key is typically handled separately or initialized as zeros/empty
    # depending on the specific training setup, but is part of the standard dictionary layout [S1, unofficial]
```

**Parameters:**
*   `model_max_length`: 77 (standard for CLIP) [S1, unofficial].
*   `embedding_dimension`: 768 (standard for CLIP ViT-L/14) [S1, unofficial].
*   `directions`: `['front', 'side', 'back']` [S1, unofficial].

**Edge Cases:**
*   **Forward-facing scenes:** For datasets like LLFF, the "back" view may be invalid or irrelevant. In such cases, the "back" prompt is omitted, and only `['front', 'side']` (and potentially `'top-down'` if applicable) are used [S2].
*   **Device Mismatch:** Ensure `input_ids` are moved to the same device as the text encoder to avoid runtime errors [S1, unofficial].

## Subtopics

### [TexGS-VolVis Representation and Optimization](11105-texgs-volvis-representation-and-optimization.md)
TexGS-VolVis is a method for representing and editing volume visualization (VolVis) scenes using textured 2D Gaussian Splatting (2DGS). It solves the problem of decoupling geometry from appearance in text-driven non-photorealistic style editing (NPSE), allowing for geometry-consistent stylization without the elliptical artifacts common in coupled representations. Unlike methods that freeze geometry to prevent distortion, TexGS-VolVis uses explicit 2D texture maps to store appearance, preserving geometric fidelity while enabling rich visual expressiveness.

### Directional Dependent Prompts
This variant concatenates a text prompt to the input caption depending on the camera location in the scene at each iteration [S2]. The prompts used are: `', top-down view'`, `', front view'`, `', side view'`, and `', back view'` [S2]. For forward-facing scenes like the LLFF dataset, only the first three captions (`top-down`, `front`, `side`) are used [S2]. This allows the model to adapt its semantic understanding based on the current viewing angle during training or rendering [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of directional views $N_d$. Encoding is performed once per prompt update or initialization.
*   **Memory Complexity:** $O(N_d \cdot L \cdot D)$, where $N_d$ is the number of directions, $L$ is sequence length (77), and $D$ is embedding dimension (768). For 4 directions, this is negligible compared to Gaussian attributes.
*   **Strengths:** Enables spatially aware text queries; simple to implement using existing CLIP libraries.
*   **Limitations:** Requires pre-defining view directions; does not inherently handle arbitrary continuous viewpoints without interpolation or additional training.

## Background
Originating from the DreamGaussian project [S1, unofficial] and Blended-NeRF [S2], this method leverages the pre-trained CLIP model to bridge 3D geometry and natural language. It was developed to support zero-shot object generation and blending in neural radiance fields and Gaussian splatting contexts, allowing users to query 3D scenes using text that respects camera orientation [S1, unofficial][S2].

## Sources
[S1] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
[S2] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
