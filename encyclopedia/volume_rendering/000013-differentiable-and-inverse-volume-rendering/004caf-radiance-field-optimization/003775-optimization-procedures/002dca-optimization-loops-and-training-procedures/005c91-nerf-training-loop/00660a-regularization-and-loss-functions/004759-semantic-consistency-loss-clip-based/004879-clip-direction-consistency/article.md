# CLIP Direction Consistency

Also called: CLIP Direction Score, CLIP directional similarity loss, Consistency Score

## Overview
CLIP Direction Consistency is a quantitative metric used to evaluate the temporal or spatial coherence of edits in Neural Radiance Fields (NeRFs) by comparing the semantic trajectory of changes between adjacent views [S1, S2]. It solves the problem of verifying that an edited scene evolves smoothly relative to the original scene, ensuring that the direction of semantic change in the edited views matches the direction of change in the original views [S2]. It is distinct from the CLIP Direction Similarity metric, which measures the alignment between the text prompt and the image features; instead, this metric focuses on the consistency of the delta (change) between consecutive frames [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of rendered images from the original NeRF ($I^o$) and a sequence of rendered images from the edited NeRF ($I^e$) [S1, S2].
*   The sequences must consist of consecutive views (e.g., adjacent camera poses) [S2].
*   **Outputs:** A scalar score representing the average cosine similarity between the change vectors of the original and edited scenes across the selected views [S1, S2].

**Data Structures:**
*   Images are treated as tensors or arrays of pixel values [G].
*   CLIP embeddings are vectors in the CLIP image feature space [S1, S2].

**Steps:**
1.  **Select View Pairs:** Identify $N$ pairs of consecutive views with indices $i$ and $i+1$ [S2].
2.  **Encode Images:** Pass the original images $I^o_i, I^o_{i+1}$ and edited images $I^e_i, I^e_{i+1}$ through the CLIP image encoder $E_I$ (or $\mathbf{C}_I$) to obtain embeddings [S1, S2].
3.  **Compute Change Vectors:** Calculate the difference between the embeddings of consecutive views for both the original and edited sequences [S2].
    *   Original change: $\Delta I^o_i = E_I(I^o_{i+1}) - E_I(I^o_i)$ [S2].
    *   Edited change: $\Delta I^e_i = E_I(I^e_{i+1}) - E_I(I^e_i)$ [S2].
4.  **Compute Cosine Similarity:** Calculate the cosine similarity between the original change vector and the edited change vector for each pair [S2].
    $$ L_{direction, i} = \frac{\Delta I^o_i \cdot \Delta I^e_i}{\|\Delta I^o_i\| \|\Delta I^e_i\|} $$
    Where $\cdot$ denotes the dot product and $\|\cdot\|$ denotes the L2 norm [S2].
5.  **Average:** Compute the final score by averaging the similarity scores over the selected consecutive view pairs [S2].
    $$ \text{Score} = \frac{1}{N} \sum_{i=1}^{N} L_{direction, i} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def clip_direction_consistency(original_images, edited_images, clip_encoder):
    """
    Computes the CLIP Direction Consistency score.
    
    Args:
        original_images (Tensor): Shape (N, C, H, W), consecutive original views.
        edited_images (Tensor): Shape (N, C, H, W), consecutive edited views.
        clip_encoder (Module): CLIP image encoder returning embeddings.
        
    Returns:
        float: Average cosine similarity of change vectors.
    """
    # Encode all frames to embeddings of shape (N, D)
    emb_o = clip_encoder(original_images)
    emb_e = clip_encoder(edited_images)
    
    # Compute change vectors (deltas) between consecutive frames
    # delta_o[i] = emb_o[i+1] - emb_o[i] for i in 0..N-2
    delta_o = emb_o[1:] - emb_o[:-1]
    delta_e = emb_e[1:] - emb_e[:-1]
    
    similarities = []
    for i in range(len(delta_o)):
        d_o = delta_o[i]
        d_e = delta_e[i]
        
        # Compute L2 norms
        norm_o = torch.norm(d_o)
        norm_e = torch.norm(d_e)
        
        # Guard against zero vectors (degenerate case where no change occurred) [G]
        if norm_o < 1e-8 or norm_e < 1e-8:
            sim = 0.0
        else:
            dot_product = torch.dot(d_o, d_e)
            sim = dot_product / (norm_o * norm_e)
        
        similarities.append(sim)
        
    if not similarities:
        return 0.0
        
    final_score = sum(similarities) / len(similarities)
    return final_score
```

**Parameters:**
*   **Number of views:** The sources specify computing the score on six consecutive views and averaging the results [S2]. This implies 5 pairs of consecutive frames [S2].

**Edge Cases and Pitfalls:**
*   **Zero Change:** If the original or edited scene does not change between two views, the delta vector is zero and the cosine similarity is undefined [G]. The implementation must handle this by skipping the pair or assigning a neutral value [G].
*   **View Ordering:** The views must be temporally or spatially ordered such that $i$ and $i+1$ are adjacent in the camera trajectory [S2]. Random pairs will yield meaningless results [G].
*   **Resolution:** The CLIP encoder expects specific input resolutions (e.g., 224x224) [G]. Images must be resized or cropped accordingly before encoding [G].

## Subtopics

### [Global-Local Contrastive Learning](13433-global-local-contrastive-learning.md)
A regularization strategy for text-driven NeRF stylization that controls stylization strength by combining global and local contrastive terms in CLIP space. It ensures uniform stylization across the entire scene, overcoming the directional-only limitation of standard CLIP losses and the non-uniformity of global-only approaches. It builds on CLIP Direction Consistency by adding magnitude control and local balance.

## Operating characteristics
**Time and Memory Complexity:**
*   Not given in the sources.

**Measured Performance:**
*   In simple settings (5 edits on face scene), ViCA-NeRF achieved a score of 0.9035 compared to Instruct-NeRF2NeRF's 0.9004 [S1].
*   In difficult settings (10 edits on real-world scenes), ViCA-NeRF achieved 0.6498 compared to Instruct-NeRF2NeRF's 0.5161 [S1].
*   Higher scores indicate better consistency between the direction of change in the original and edited scenes [S2].

**Strengths and Limitations:**
*   **Strengths:** Provides a semantic measure of edit consistency that correlates with human perception of smoothness [S2].
*   **Limitations:** It is a post-hoc evaluation metric, not a loss function used during training [S2].

**When to Use:**
*   Use for evaluating the quality of NeRF editing methods, specifically when assessing the smoothness and coherence of semantic changes across views [S2].

## Background
The CLIP Direction Consistency metric was introduced in the context of evaluating neural radiance field editing methods [S1, S2]. It appears in ViCA-NeRF [S1] and Blended-NeRF [S2] for evaluation purposes [S1, S2].

## Sources
[S1] ViCA-NeRF: View-Consistency-Aware 3D Editing of Neural Radiance Fields, https://arxiv.org/abs/2402.00864v1, fetched 2026-10-26
[S2] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
