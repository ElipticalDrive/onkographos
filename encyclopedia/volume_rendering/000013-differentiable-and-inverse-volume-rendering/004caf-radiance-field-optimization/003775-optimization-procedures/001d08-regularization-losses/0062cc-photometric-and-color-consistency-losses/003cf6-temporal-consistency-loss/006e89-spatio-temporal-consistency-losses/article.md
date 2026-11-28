# Spatio-Temporal Consistency Losses

Also called: Consistency Learning, Lbg, Lconsist, Lid, Spatio-Temporal Consistency Constraints

## Overview
Spatio-temporal consistency losses are a set of regularization terms used in differentiable radiance field optimization to prevent flickering and attribute drift when synthesizing dynamic content from static identity representations. Unlike [Temporal Consistency Loss](9372-temporal-consistency-loss.md), which enforces coherence between adjacent time steps, this method enforces invariance of subject-specific attributes (identity, albedo, shape) and background regions across different motion conditions generated from the same identity latent code. It is distinct from [Scene Flow Regularization](18522-scene-flow-regularization.md) by focusing on semantic attribute preservation rather than geometric flow continuity.

## How it works
The method operates within a generative radiance field framework (specifically CoRF [S1]) where a generator $G$ renders images from 3D features. To ensure that changes in motion do not alter the identity or background, the method generates two images using the same identity representation but different motion representations, then constrains their derived attributes to be identical.

### Inputs and Outputs
*   **Inputs:**
    *   Identity latent code $z$ (noise vector).
    *   Two distinct motion representations $m_1$ and $m_2$.
    *   Camera parameters (origin $o$, ray direction $d$).
*   **Outputs:**
    *   Two synthetic images $I_0^1$ and $I_0^2$.
    *   Loss values $L_{consist}$, $L_{id}$, and $L_{bg}$.

### Data Structures
The method relies on pre-trained external networks to extract features. The internal data structures are standard image tensors and latent vectors.
*   **Images:** $I \in \mathbb{R}^{H \times W \times 3}$ (RGB).
*   **Latents:** $z \in \mathbb{R}^D$ (identity noise), $m \in \mathbb{R}^K$ (motion).
*   **Attributes:** Vectors $l, a, t, s$ representing lighting, albedo, texture, and shape, respectively, extracted by a head regressor $R$.
*   **Identity Embeddings:** Vectors $e \in \mathbb{R}^E$ extracted by an identity encoder $E$.
*   **Masks:** Binary masks $s_1, s_2 \in \{0, 1\}^{H \times W}$ indicating background regions (1) vs. face regions (0), extracted by a face parsing net.

### Steps
1.  **Generate Paired Images:** Render two images $I_0^1$ and $I_0^2$ using the generator $G$. Both images use the same identity noise $z$ but different motion inputs $m_1$ and $m_2$.
    *   The generator integrates 3D features along rays using the rendering equation [S1]:
        $$ F_r = \int_{t_n}^{t_f} T(t) \sigma(\mathbf{r}(t)) \mathbf{F}(\mathbf{r}(t), d, m, z) dt $$
        where $T(t) = \exp\left(-\int_{t_n}^{t} \sigma(\mathbf{r}(s), m, z) ds\right)$.
2.  **Extract Attributes:** Apply a pre-trained head regressor $R$ to both $I_0^1$ and $I_0^2$ to decompose them into attribute parameters: lighting ($l$), albedo ($a$), texture ($t$), and shape ($s$).
    *   Let $R(I_0^1) = \{l_1, a_1, t_1, s_1^{attr}\}$ and $R(I_0^2) = \{l_2, a_2, t_2, s_2^{attr}\}$. Note: $s_1^{attr}$ and $s_2^{attr}$ refer to shape attributes, not masks.
3.  **Compute Attribute Consistency Loss ($L_{consist}$):** Compare the extracted attributes. The set of attribute pairs is $A = \{(l_1, l_2), (a_1, a_2), (t_1, t_2), (s_1^{attr}, s_2^{attr})\}$.
    $$ L_{consist} = \mathbb{E}_{I_0 \sim p_{syn}} \sum_{(x_1, x_2) \in A} \beta_x \|x_1 - x_2\|_2^2 $$
    where $\beta_x$ is a loss weight for attribute $x$ [S1].
4.  **Extract Identity Embeddings:** Apply a pre-trained identity encoder $E$ to both images to get embeddings $e_1 = E(I_0^1)$ and $e_2 = E(I_0^2)$.
5.  **Compute Identity Loss ($L_{id}$):** Minimize the distance between embeddings, defined as negative cosine similarity.
    $$ L_{id} = \mathbb{E}_{I_0^1, I_0^2} \left( 1 - \frac{e_1 \cdot e_2}{\|e_1\|_2 \cdot \|e_2\|_2} \right) $$
6.  **Extract Background Masks:** Apply a pre-trained face parsing net to $I_0^1$ and $I_0^2$ to obtain binary masks $m_1^{bg}$ and $m_2^{bg}$ (using $s_1, s_2$ in source notation). Values are 1 for background, 0 for face.
7.  **Compute Background Consistency Loss ($L_{bg}$):** Compare pixel values only in the overlapping background regions.
    $$ L_{bg} = \mathbb{E}_{I_0^1, I_0^2} \frac{\| (m_1^{bg} \odot m_2^{bg}) \odot (I_0^1 - I_0^2) \|_2}{\| m_1^{bg} \odot m_2^{bg} \|} $$
    where $\odot$ denotes element-wise product [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spatio_temporal_consistency_loss(
    generator, 
    regressor, 
    identity_encoder, 
    face_parsing_net,
    z,          # Identity noise vector
    m1,         # Motion representation 1
    m2,         # Motion representation 2
    camera_params, # Includes ray origins and directions
    beta_weights # Dict mapping attribute names to beta_x weights
):
    # Step 1: Generate two images with same identity, different motion
    I_0_1 = generator.render(z=z, motion=m1, camera=camera_params)
    I_0_2 = generator.render(z=z, motion=m2, camera=camera_params)
    
    # Step 2: Extract attributes using head regressor
    # Returns dict with keys: 'lighting', 'albedo', 'texture', 'shape'
    attrs_1 = regressor.predict(I_0_1)
    attrs_2 = regressor.predict(I_0_2)
    
    # Step 3: Compute Attribute Consistency Loss
    L_consist = 0.0
    for attr_name in ['lighting', 'albedo', 'texture', 'shape']:
        x1 = attrs_1[attr_name]
        x2 = attrs_2[attr_name]
        beta = beta_weights.get(attr_name, 1.0)
        L_consist += beta * torch.mean((x1 - x2) ** 2)
        
    # Step 4: Extract Identity Embeddings
    e1 = identity_encoder(I_0_1)
    e2 = identity_encoder(I_0_2)
    
    # Step 5: Compute Identity Loss (Negative Cosine Similarity)
    # Add small epsilon to avoid division by zero
    eps = 1e-8
    dot_product = torch.sum(e1 * e2)
    norm1 = torch.norm(e1) + eps
    norm2 = torch.norm(e2) + eps
    cosine_sim = dot_product / (norm1 * norm2)
    L_id = 1.0 - cosine_sim
    
    # Step 6: Extract Background Masks
    # face_parsing_net outputs binary mask: 1 for background, 0 for face
    mask1 = face_parsing_net.get_background_mask(I_0_1)
    mask2 = face_parsing_net.get_background_mask(I_0_2)
    
    # Step 7: Compute Background Consistency Loss
    # Element-wise product to find overlapping background regions
    overlap_mask = mask1 * mask2
    
    # Difference in pixel values
    diff = I_0_1 - I_0_2
    
    # Weighted difference norm
    numerator = torch.norm(overlap_mask * diff)
    denominator = torch.norm(overlap_mask)
    
    # Avoid division by zero if no background overlap
    if denominator > eps:
        L_bg = numerator / denominator
    else:
        L_bg = 0.0
        
    return L_consist, L_id, L_bg
```

### Parameters
*   **$\lambda_1 \dots \lambda_5$:** Hyper-parameters adjusting the strength of adversarial, motion, consistency, identity, and background losses respectively [S1].
*   **$\beta_x$:** Loss weights for specific attributes in $L_{consist}$ [S1].
*   **Resolution:** Typically $256 \times 256$ in the source evaluation [S1].

### Edge Cases and Pitfalls
*   **No Background Overlap:** If the motion causes the face to occlude the entire background or shift entirely out of frame, the denominator in $L_{bg}$ may be zero. A guard clause is required [G].
*   **Pre-trained Network Drift:** The method relies on $R$, $E$, and the parsing net being pre-trained. If these networks are not aligned with the generator's output distribution, the consistency constraints may fail or introduce artifacts [S1].
*   **Identity Leakage:** If the motion representation $m$ contains identity information, $L_{id}$ and $L_{consist}$ may conflict with motion control objectives. The source notes that motion is embedded into a layered latent space to mitigate this [S1].

## Operating characteristics
*   **Time Complexity:** Adds overhead proportional to the number of forward passes through the regressor, identity encoder, and parsing net per training step. These are typically lightweight compared to the radiance field rendering itself, but scale with image resolution.
*   **Memory Complexity:** Requires storing intermediate feature maps for two images and their derived attributes/embeddings. No significant additional memory structures beyond standard training buffers.
*   **Performance:** Reduces flickering and attribute drift in dynamic face synthesis. Evaluated on FFHQ and face video benchmarks at $256 \times 256$ [S1].
*   **Strengths:** Explicitly decouples identity/background from motion, ensuring stable appearance over time.
*   **Limitations:** Requires pre-trained auxiliary networks (regressor, encoder, parser). Does not handle non-face content or general scene consistency without adaptation.

## Background
The method was introduced in "Controllable Radiance Fields for Dynamic Face Synthesis" (CoRF) by Yadira et al. in 2022 [S1]. It addresses the lack of ground-truth targets for novel identities in generative radiance fields by using self-supervised consistency constraints. It builds on StyleNeRF [S1] and prior face reenactment work [S1].

## Sources
[S1] Controllable Radiance Fields for Dynamic Face Synthesis, https://doi.org/10.1109/3dv57658.2022.00075, fetched 2026-10-29
