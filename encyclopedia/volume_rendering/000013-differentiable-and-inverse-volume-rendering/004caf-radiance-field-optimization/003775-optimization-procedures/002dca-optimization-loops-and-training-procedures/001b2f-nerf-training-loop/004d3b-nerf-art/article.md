# NeRF-Art

Also called: Text-Driven Neural Radiance Fields Stylization

## Overview
NeRF-Art is a text-guided method for stylizing pre-trained Neural Radiance Fields (NeRFs) by simultaneously altering both appearance and geometry without requiring mesh guidance [S1]. It distinguishes itself from prior approaches by employing a hybrid global-local contrastive learning strategy based on CLIP, which ensures strong style adherence and cross-view consistency, addressing the insufficiency of direct CLIP similarity constraints [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model (representing the original scene's geometry and appearance) and a text prompt describing the target style [S1].
*   **Output:** A stylized NeRF model where the density field and color field are modified to reflect the target style's geometry and appearance [S1].

**Data Structures:**
The method operates on the standard NeRF representation:
*   **Density Field ($\sigma$):** A multi-layer perceptron (MLP) mapping 3D coordinates to volume density.
*   **Color Field ($c$):** An MLP mapping 3D coordinates and viewing direction to RGB color.
*   **CLIP Encoder:** Used to map rendered images and text prompts into a shared embedding space [S1].

**Parameters:**
*   **Negative Samples:** A set of pre-defined styles used as negative examples in the contrastive loss [S1].
*   **Weight Regularization Coefficient:** A parameter to suppress cloudy artifacts and geometry noises during density field transformation [S1].
*   **Learning Rate:** For optimizing the NeRF weights [S1].

**Numbered Steps:**

1.  **Initialization:** Load the pre-trained NeRF model and the CLIP model [S1].
2.  **Ray Marching:** For each training camera pose, generate rays and render images using the current NeRF parameters. This involves sampling points along the ray and integrating density and color using standard volume rendering equations [S1].
3.  **Embedding Extraction:**
    *   Encode the target text prompt into a text embedding vector using the CLIP text encoder [S1].
    *   Encode the rendered image into an image embedding vector using the CLIP image encoder [S1].
    *   Encode the negative style samples into their respective embedding vectors [S1].
4.  **Contrastive Loss Calculation:** Compute a CLIP-based contrastive loss to strengthen stylization. This loss brings the rendered image embedding closer to the target text embedding and pushes it away from the negative style embeddings [S1].
5.  **Hybrid Global-Local Framework:** Extend the contrastive constraint to cover both global structures and local details. This ensures uniformity of the style over the whole scene. The specific implementation involves computing contrastive losses at different scales or regions to capture both coarse and fine style attributes [S1].
6.  **Density Field Relaxation and Regularization:** To support geometry stylization jointly with appearance, relax the constraints on the density of the pre-trained NeRF. Apply weight regularization to the density field parameters to effectively suppress cloudy artifacts and geometry noises that arise easily when the density field is transformed [S1].
7.  **Optimization:** Update the NeRF weights (density and color MLP parameters) using gradient descent to minimize the combined loss (contrastive loss + regularization) [S1].
8.  **Iteration:** Repeat steps 2-7 for a specified number of epochs or until convergence [S1].

**Formulas:**
*   **Contrastive Loss:** The method uses a contrastive loss to maximize the similarity between the rendered view and the target text, while minimizing similarity with negative samples. Let $E_I(I)$ be the image embedding of the rendered image $I$, $E_T(T)$ be the text embedding of the target text $T$, and $E_T(N_i)$ be the text embeddings of negative samples $N_i$. The loss encourages $\cos(E_I(I), E_T(T))$ to be high and $\cos(E_I(I), E_T(N_i))$ to be low [S1].
*   **Weight Regularization:** Applied to the density field weights to prevent noisy geometry [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_art_stylization(pretrained_nerf, text_prompt, negative_styles, num_epochs, learning_rate):
    """
    Stylize a pre-trained NeRF using text guidance and contrastive learning.
    
    Args:
        pretrained_nerf: The initial NeRF model (density and color MLPs).
        text_prompt: String describing the target style.
        negative_styles: List of strings for negative style samples.
        num_epochs: Number of training iterations.
        learning_rate: Step size for optimizer.
        
    Returns:
        stylized_nerf: The optimized NeRF model.
    """
    # Initialize CLIP models
    clip_text_encoder = load_clip_text_encoder()
    clip_image_encoder = load_clip_image_encoder()
    
    # Optimize NeRF weights
    optimizer = torch.optim.Adam(pretrained_nerf.parameters(), lr=learning_rate)
    
    for epoch in range(num_epochs):
        optimizer.zero_grad()
        
        # 1. Render images from current NeRF
        # Assume get_rendered_image returns the rendered RGB image for a given camera pose
        rendered_image = get_rendered_image(pretrained_nerf, camera_pose)
        
        # 2. Extract embeddings
        target_text_embedding = clip_text_encoder(text_prompt)
        rendered_image_embedding = clip_image_encoder(rendered_image)
        
        negative_embeddings = []
        for neg_style in negative_styles:
            neg_text_embedding = clip_text_encoder(neg_style)
            negative_embeddings.append(neg_text_embedding)
            
        # 3. Compute Contrastive Loss
        # Loss brings rendered image closer to target text and farther from negatives
        contrastive_loss = compute_contrastive_loss(
            rendered_image_embedding, 
            target_text_embedding, 
            negative_embeddings
        )
        
        # 4. Compute Hybrid Global-Local Loss
        # Ensures style uniformity across global structures and local details
        hybrid_loss = compute_hybrid_global_local_loss(
            pretrained_nerf, 
            rendered_image, 
            target_text_embedding
        )
        
        # 5. Compute Weight Regularization Loss
        # Suppresses cloudy artifacts and geometry noises in density field
        regularization_loss = compute_density_weight_regularization(pretrained_nerf)
        
        # 6. Combine Losses
        total_loss = contrastive_loss + hybrid_loss + regularization_loss
        
        # 7. Backpropagate and Update
        total_loss.backward()
        optimizer.step()
        
    return pretrained_nerf

def compute_contrastive_loss(image_emb, target_text_emb, negative_text_embs):
    """
    Computes contrastive loss to maximize similarity with target and minimize with negatives.
    """
    # Cosine similarity between image and target text
    sim_target = cosine_similarity(image_emb, target_text_emb)
    
    # Cosine similarities between image and negative texts
    sim_negatives = [cosine_similarity(image_emb, neg_emb) for neg_emb in negative_text_embs]
    
    # Contrastive loss formulation (e.g., InfoNCE or similar contrastive objective)
    # The exact formula is not provided in the source, but the goal is to maximize sim_target
    # and minimize sim_negatives.
    # Placeholder for the specific contrastive loss function implementation
    loss = -sim_target + sum(sim_negatives) # Simplified representation
    return loss

def compute_hybrid_global_local_loss(nerf, rendered_image, target_text_emb):
    """
    Computes loss to ensure style uniformity over global structures and local details.
    """
    # This involves extending the contrastive constraint to cover both global and local aspects.
    # Specific implementation details for global vs local feature extraction are not provided.
    # Placeholder for the hybrid loss calculation
    loss = 0.0 # Placeholder
    return loss

def compute_density_weight_regularization(nerf):
    """
    Applies weight regularization to the density field to suppress artifacts.
    """
    # Access density MLP weights
    density_weights = nerf.density_mlp.parameters()
    
    # Apply regularization (e.g., L2 norm or specific weight constraint)
    # The exact regularization term is not specified in the source.
    regularization = sum(torch.norm(w) for w in density_weights) # Placeholder
    return regularization
```

**Edge Cases and Pitfalls:**
*   **Cloudy Artifacts and Geometry Noises:** These arise easily when the density field is transformed during geometry stylization. This is handled by weight regularization [S1].
*   **Insufficient Style Strength:** Directly applying supervision from CLIP by constraining similarity between rendered views and text is insufficient. The contrastive loss with negative samples is required to ensure desired style strength [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method demonstrates effectiveness and robustness regarding both single-view stylization quality and cross-view consistency across various styles [S1]. It achieves the best visual-pleasing results compared to related methods in user studies [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Simultaneously alters appearance and geometry; requires no mesh guidance; effective text guidance; robust cross-view consistency [S1].
*   **Limitations:** Requires careful handling of density field transformation to avoid artifacts; relies on pre-defined negative samples for contrastive learning [S1].
*   **When to use:** When text-driven stylization of both appearance and geometry is desired without mesh availability [S1].
*   **When not:** When mesh guidance is available and preferred, or when real-time performance is critical and the computational cost of CLIP integration is prohibitive [S1].

## Background
NeRF-Art was introduced by Can Wang, Ruixiang Jiang, Mingming He, Dongdong Chen, and Jing Liao [S1]. It addresses the challenge of stylizing NeRFs with text prompts, a task that is non-trivial due to the need to simultaneously control appearance and geometry [S1]. The method builds on the integration of large-scale Language-Vision models (CLIP) with NeRF [S1].

## Sources
[S1] NeRF-Art: Text-Driven Neural Radiance Fields Stylization, https://doi.org/10.1109/tvcg.2023.3283400, fetched 2026-10-26
