# CLIP Direction Consistency

Also called: CLIP Direction Score

## Overview
CLIP Direction Consistency is a regularization metric used in text-driven NeRF stylization to ensure that the semantic trajectory of the edited scene matches that of the original scene across adjacent camera views [S1]. It solves the problem of non-uniform stylization by enforcing that the direction of change in CLIP feature space between consecutive frames remains consistent between the original and edited renderings [S1]. Unlike standard CLIP losses that only measure absolute similarity to a text prompt, this method measures the *relative* consistency of style evolution, preventing artifacts where stylization strength varies unpredictably across the scene [S1].

## How it works
**Inputs and Outputs**
- **Inputs**: A sequence of $N$ consecutive rendered views from the original scene ($I^o_1, \dots, I^o_N$) and the corresponding edited/stylized scene views ($I^e_1, \dots, I^e_N$). The views are typically sampled from adjacent camera positions in the training trajectory.
- **Outputs**: A scalar loss value $L_{direction}$ representing the average cosine similarity between the change vectors of the original and edited scenes.

**Data Structures**
- $E_I(\cdot)$: The CLIP image encoder (typically ViT-L/14 or ViT-B/16) which maps an image to a normalized embedding vector in the CLIP latent space.
- $I^o_i$: Original view at index $i$.
- $I^e_i$: Edited view at index $i$.

**Steps**
1. **Select View Pairs**: Choose a set of consecutive view indices. The source specifies computing the score on six consecutive views and averaging the results [S1]. Let the indices be $i$ and $i+1$ for $i = 1$ to $5$ (assuming 6 views total).
2. **Encode Original Views**: Compute CLIP embeddings for the original consecutive views:
   $$ E^o_i = E_I(I^o_i) $$
   $$ E^o_{i+1} = E_I(I^o_{i+1}) $$
3. **Encode Edited Views**: Compute CLIP embeddings for the edited consecutive views:
   $$ E^e_i = E_I(I^e_i) $$
   $$ E^e_{i+1} = E_I(I^e_{i+1}) $$
4. **Compute Change Vectors**: Calculate the difference vectors in the CLIP embedding space for both original and edited sequences:
   $$ \Delta I^o_i = E^o_{i+1} - E^o_i $$
   $$ \Delta I^e_i = E^e_{i+1} - E^e_i $$
5. **Compute Cosine Similarity**: Calculate the cosine similarity between the change vectors for each pair:
   $$ S_i = \frac{\Delta I^o_i \cdot \Delta I^e_i}{\|\Delta I^o_i\| \|\Delta I^e_i\|} $$
   where $\cdot$ denotes the dot product and $\|\cdot\|$ denotes the L2 norm.
6. **Average**: Compute the final loss by averaging the similarities over the selected consecutive pairs:
   $$ L_{direction} = \frac{1}{K} \sum_{i=1}^{K} S_i $$
   where $K$ is the number of pairs (e.g., $K=5$ for 6 views).

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clip_direction_consistency(original_views, edited_views, clip_encoder):
    """
    Computes the CLIP Direction Consistency loss.
    
    Args:
        original_views: List of N original rendered images [I^o_1 ... I^o_N]
        edited_views: List of N edited rendered images [I^e_1 ... I^e_N]
        clip_encoder: Function E_I that returns CLIP embeddings for an image
        
    Returns:
        Scalar loss value L_direction
    """
    # Ensure we have at least 2 views
    if len(original_views) < 2:
        return 0.0

    similarities = []
    
    # Iterate through consecutive pairs
    for i in range(len(original_views) - 1):
        # Step 1: Encode original views
        emb_o_curr = clip_encoder(original_views[i])
        emb_o_next = clip_encoder(original_views[i+1])
        
        # Step 2: Encode edited views
        emb_e_curr = clip_encoder(edited_views[i])
        emb_e_next = clip_encoder(edited_views[i+1])
        
        # Step 3: Compute change vectors
        delta_o = emb_o_next - emb_o_curr
        delta_e = emb_e_next - emb_e_curr
        
        # Step 4: Compute norms
        norm_o = torch.linalg.norm(delta_o)
        norm_e = torch.linalg.norm(delta_e)
        
        # Handle degenerate cases where change is zero
        if norm_o < 1e-8 or norm_e < 1e-8:
            continue
            
        # Step 5: Compute cosine similarity
        dot_product = torch.dot(delta_o, delta_e)
        similarity = dot_product / (norm_o * norm_e)
        
        similarities.append(similarity)
        
    # Step 6: Average
    if not similarities:
        return 0.0
        
    return torch.mean(torch.tensor(similarities))
```

**Parameters**
- **Number of views**: The source specifies using six consecutive views [S1].
- **CLIP Model**: The specific variant (e.g., ViT-L/14) is not explicitly detailed in the formula section, but standard CLIP image encoders are implied [S1].

**Edge Cases and Pitfalls**
- **Zero Change**: If the CLIP embeddings for consecutive frames are identical (e.g., static camera or no change in stylization), the denominator becomes zero. A guard clause or small epsilon should be added to prevent division by zero [G].
- **View Selection**: The metric assumes a temporal or spatial continuity in the camera trajectory. Randomly sampled views may not represent a valid "direction" of change [S1].

## Subtopics

### [Global-Local Contrastive Learning](13433-global-local-contrastive-learning.md)
Extends directional consistency by adding local contrastive terms to ensure uniform stylization magnitude, addressing the limitation that directional CLIP losses alone may fail to enforce sufficient stylization strength across the entire scene.

## Operating characteristics
- **Time Complexity**: Dominated by the CLIP encoder inference. For $N$ views, it requires $2N$ forward passes through the CLIP image encoder. This is significantly more expensive than photometric losses but cheaper than full NeRF training steps.
- **Memory Complexity**: Requires storing embeddings for $2N$ views. Since CLIP embeddings are fixed-size vectors (e.g., 768 or 1024 dimensions), memory overhead is low compared to storing full images or volumes.
- **Strengths**: Enforces semantic consistency in the *change* rather than just the final state, leading to more coherent stylization trajectories [S1].
- **Limitations**: Relies on the quality of CLIP embeddings; may not capture fine-grained geometric details or texture changes that do not affect high-level semantics [S1].
- **When to use**: Text-driven NeRF stylization where maintaining the relative style evolution between original and edited scenes is critical [S1].

## Background
The CLIP Direction Score was introduced in [23] (cited within [S1]) as a metric to evaluate consistency between adjacent frames in edited NeRF scenes. It was adopted in Blended-NeRF [S1] to regularize the blending of generated objects into existing radiance fields. The method leverages the semantic alignment capabilities of CLIP to ensure that the "direction" of the edit in feature space is preserved across the scene.

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
