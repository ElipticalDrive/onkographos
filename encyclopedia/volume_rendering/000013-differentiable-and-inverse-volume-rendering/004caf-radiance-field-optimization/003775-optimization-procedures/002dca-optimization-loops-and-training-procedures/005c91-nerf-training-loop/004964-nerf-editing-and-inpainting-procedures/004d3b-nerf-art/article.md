# NeRF-Art

Also called: Text-Driven Neural Radiance Fields Stylization

## Overview
NeRF-Art is a text-guided method for stylizing pre-trained Neural Radiance Fields (NeRFs) that simultaneously alters both the appearance and the 3D geometry of a scene without requiring mesh guidance [S1]. It distinguishes itself from prior approaches by introducing a global-local contrastive learning strategy combined with directional constraints, addressing the insufficiency of direct CLIP similarity supervision in ensuring strong style adherence and cross-view consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model $f_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$, where $\mathbf{x}$ is 3D position, $\mathbf{d}$ is viewing direction, $\sigma$ is volume density, and $\mathbf{c}$ is color [Parent]. A target style text prompt $T_{\text{target}}$ and a set of negative style text prompts $\{T_{\text{neg}}^k\}$ [S1].
*   **Output:** Optimized NeRF parameters $\theta'$ that render images matching the target style in both texture and geometry [S1].

**Data Structures:**
The method operates on the standard NeRF MLP representation [Parent]. No additional explicit data structures (like grids or meshes) are introduced for the stylization process itself, though the method supports mesh extraction from the final $\sigma$ field for evaluation [S1].

**Optimization Objective:**
The optimization minimizes a total loss combining photometric reconstruction, a hybrid global-local contrastive style loss, and a weight regularization term [S1, Parent].
1.  **Reconstruction Loss:** Standard photometric loss on visible pixels to preserve identity where style is not applied [S1].
2.  **Style Loss:** A hybrid global-local contrastive loss based on CLIP embeddings [S1].
3.  **Regularization:** Weight regularization to suppress artifacts during geometry stylization [S1].

**Step-by-Step Method:**

1.  **Ray Marching:** For each camera ray, sample points along the ray and compute transmittance and density using the current NeRF $f_\theta$ [G]. Render the image using the standard volume rendering equation [G].
2.  **CLIP Embedding Extraction:** Pass the rendered image and text prompts through the CLIP image encoder $E_I$ and text encoder $E_T$ [S1]. Extract the target embedding from the target style prompt, and negative embeddings from pre-defined negative style prompts [S1]. Extract the rendered embedding from the generated image [S1].
3.  **Global-Local Contrastive Loss Calculation:** The style loss is defined as a combination of global and local contrastive terms [S1]. The global contrastive loss measures similarity between the entire rendered image and the target text, while pushing away from negative texts [S1]. To ensure uniformity of style over local details, the contrastive constraint is extended to a local framework by dividing the image into patches [S1]. A directional constraint is applied to control the trajectory and strength of the style, ensuring consistent application across views [S1, S2].
4.  **Geometry Stylization and Regularization:** To support joint geometry and appearance stylization, constraints on the density field are relaxed [S1]. Cloudy artifacts and geometry noise arise easily when the density field is transformed during geometry stylization [S1]. A weight regularization term is applied to the NeRF weights to suppress these artifacts [S1].
5.  **Optimization Loop:** Update $\theta$ using gradient descent to minimize the combined loss [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_art_stylization(nerf_params, target_text, negative_texts, ref_images, cameras, 
                         steps, lr, lambda_rec, lambda_style, lambda_reg):
    """
    Stylizes a pre-trained NeRF using text-driven contrastive learning.
    """
    # Initialize optimizer (standard gradient descent with momentum) [G]
    optimizer = Adam(nerf_params, lr=lr)
    # Pre-compute CLIP text embeddings [G]
    clip_text_enc = load_clip_text_encoder()
    target_emb = clip_text_enc.encode(target_text)
    neg_embs = [clip_text_enc.encode(txt) for txt in negative_texts]

    for _ in range(steps):
        optimizer.zero_grad()
        total_loss = 0.0

        # 1. Render images
        rendered_imgs = []
        for cam in cameras:
            rays = generate_rays(cam)
            # Standard volume rendering [G]
            img = volume_render(nerf_params, rays)
            rendered_imgs.append(img)

        # 2. Compute Reconstruction Loss
        if ref_images:
            total_loss += lambda_rec * mse_loss(rendered_imgs, ref_images)

        # 3. Compute Style Loss (Global-Local Contrastive)
        style_loss = 0.0
        img_embs = [load_clip_image_encoder().encode(img) for img in rendered_imgs]

        for img_emb in img_embs:
            # Global term
            sim_target = cosine_sim(img_emb, target_emb)
            sim_negs = [cosine_sim(img_emb, neg) for neg in neg_embs]
            # Contrastive loss formulation [G]
            exp_target = exp(sim_target)
            exp_negs = sum(exp(s) for s in sim_negs)
            style_loss += -log(exp_target / (exp_target + exp_negs))

        # Local term (patch-based) [G]
        # style_loss += compute_local_contrastive(rendered_imgs, target_emb, neg_embs)

        total_loss += lambda_style * style_loss

        # 4. Compute Weight Regularization
        total_loss += lambda_reg * weight_reg_loss(nerf_params)

        # 5. Backpropagate and Update
        total_loss.backward()
        optimizer.step()
        
    return nerf_params
```

**Parameters:**
*   **Negative Prompts:** Pre-defined styles used as negative samples [S1].
*   **Loss Weights ($\lambda_{\text{rec}}, \lambda_{\text{style}}, \lambda_{\text{reg}}$):** Balances reconstruction, style, and regularization [S1].
*   **Temperature Parameter:** Controls the sharpness of the contrastive loss. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Cloudy Artifacts:** Transforming the density field for geometry stylization can easily introduce cloudy artifacts and geometry noise; handled by weight regularization [S1].
*   **Insufficient Style Strength:** Direct CLIP similarity supervision is insufficient to ensure desired style strength; addressed with contrastive loss [S1].
*   **View Consistency:** Without proper constraints, style may vary inconsistently across views; enforced via the global-local framework and directional constraints [S1, S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Simultaneous geometry and appearance stylization without mesh guidance; strong style adherence via contrastive learning; cross-view consistency [S1].
*   **Limitations:** Requires careful tuning of regularization to avoid artifacts; dependent on CLIP's style understanding; slower than non-differentiable stylization methods [S1].
*   **When to use:** When text-driven stylization of both geometry and appearance is needed without mesh data [S1].
*   **When not to use:** When real-time performance is critical or when mesh-based stylization is preferred.

## Background
NeRF-Art was introduced by Can Wang, Ruixiang Jiang, Mingming He, Dongdong Chen, and Jing Liao [S1]. It addresses the challenge of text-guided NeRF stylization, which is difficult due to the need for simultaneous geometry and appearance transformation [S1]. The method builds on the use of large-scale Language-Vision models (CLIP) with NeRF [S1].

## Sources
[S1] NeRF-Art: Text-Driven Neural Radiance Fields Stylization, https://doi.org/10.1109/tvcg.2023.3283400, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
