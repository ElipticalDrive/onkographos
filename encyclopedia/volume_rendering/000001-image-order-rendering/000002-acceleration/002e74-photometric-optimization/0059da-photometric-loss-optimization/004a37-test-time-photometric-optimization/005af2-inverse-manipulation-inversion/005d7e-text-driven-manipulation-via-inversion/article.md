# Text-driven Manipulation via Inversion

Also called: CLIP-Driven Inverse Manipulation, Vanilla Inversion

## Overview
A method to find an optimal latent code $w$ by gradient-based optimization that maximizes the cosine similarity between rendered images and a target text embedding in CLIP space [S1]. It sets itself apart from sibling methods by directly optimizing the latent representation against a text prompt rather than reconstructing a specific input image or interpolating between fixed codes [S1].

## How it works
**Inputs:**
*   A trained scene manipulator $G$ (neural radiance field or similar generative model).
*   A set of valid camera poses $[R|t]$.
*   A target text prompt $p$.
*   An initial latent code $w$ (often from the learned manifold $\bar{W}$).

**Outputs:**
*   An optimized latent code $w^*$ that generates images reflecting the visual attributes of $p$ [S1].

**Data Structures:**
*   $w$: The latent code vector being optimized. Its dimensionality is determined by the generator $G$ [S1].
*   $I^{G,w}_{[R|t]}$: The rendered image(s) produced by $G$ using code $w$ and camera poses $[R|t]$ [S1].
*   $p$: The text prompt, encoded into a feature vector by a pretrained CLIP model [S1].

**Steps:**
1.  Render images $I^{G,w}_{[R|t]}$ from the current latent code $w$ using the generator $G$ and camera poses $[R|t]$ [S1].
2.  Extract the text embedding for the target prompt $p$ using the pretrained CLIP model [S1].
3.  Compute the cosine similarity $D_{CLIP}$ between the rendered image features and the text embedding [S1].
4.  Calculate the loss $L_{CLIP}$ [S1]:
    $$ L_{CLIP} = 1 - D_{CLIP}(I^{G,w}_{[R|t]}, p) $$
5.  Perform gradient descent on $w$ to minimize $L_{CLIP}$ [S1].
6.  Repeat steps 1-5 until convergence to obtain $w^*$ [S1]:
    $$ w^* = \arg\max_w D_{CLIP}(I^{G,w}_{[R|t]}, p) $$

**Parameters:**
*   Learning rate for $w$: Not given in the sources.
*   Number of iterations: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Linked Local Attribute Problem:** Vanilla inversion suffers from a limitation where interpolations between learned latent codes cause facial attributes in different locations to change simultaneously. This prevents independent control of local features (e.g., opening eyes while keeping the mouth closed) because the method is confined to represent deformations equivalent to those from the learned manifold $\bar{W}$ [S1].
*   **Sub-optimality:** The method is described as sub-optimal due to the linked attribute problem [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def text_driven_manipulation_via_inversion(G, camera_poses, text_prompt, initial_w, clip_model, optimizer_config):
    """
    Optimizes latent code w to maximize CLIP similarity with text prompt.
    
    Args:
        G: Trained scene manipulator (neural radiance field).
        camera_poses: Set of valid camera poses [R|t].
        text_prompt: Target text string.
        initial_w: Initial latent code vector.
        clip_model: Pretrained CLIP model for text and image embedding.
        optimizer_config: Configuration for the optimizer (e.g., learning rate).
        
    Returns:
        w_star: Optimized latent code.
    """
    # Initialize w
    w = initial_w.clone().requires_grad_(True)
    
    # Initialize optimizer
    optimizer = torch.optim.Adam([w], **optimizer_config)
    
    # Pre-compute text embedding
    text_embedding = clip_model.encode_text(text_prompt)
    
    # Optimization loop
    for iteration in range(max_iterations): # Not given in sources
        optimizer.zero_grad()
        
        # Render images using current w and camera poses
        # G returns rendered images I^{G,w}_{[R|t]}
        rendered_images = G(w, camera_poses)
        
        # Get image embeddings from CLIP
        # Note: CLIP typically processes single images; 
        # if multiple poses, average or select best similarity.
        # Assuming rendered_images is a batch or single image.
        image_embedding = clip_model.encode_image(rendered_images)
        
        # Compute cosine similarity
        # D_CLIP is cosine similarity between image_embedding and text_embedding
        similarity = cosine_similarity(image_embedding, text_embedding)
        
        # Compute loss: L_CLIP = 1 - D_CLIP
        loss = 1.0 - similarity
        
        # Backpropagate
        loss.backward()
        
        # Update w
        optimizer.step()
        
    return w.detach()
```

## Subtopics

### Face Reconstruction via Latent Code Optimization
A variant that solves an optimization problem to find a latent code that produces a good starting point for face geometry reconstruction, given K views of a target identity [S2]. It differs from the text-driven method by optimizing for perceptual and reconstruction losses against image patches rather than text similarity [S2]. It is chosen when the goal is to reconstruct a specific identity's geometry from sparse views rather than manipulating attributes via text [S2]. The optimization is performed at the same resolution as the prior model after removing the background [S2]. The latent code $w_{target}$ is recovered by minimizing:
$$ \mathbf {w}_\text {target} = \argmin _{\mathbf {w}} \mathcal {L}_\text {recon} + \lambda _\text {LPIPS} \mathcal {L}_\text {LPIPS} $$
where $\mathcal {L}_\text {recon} = \| \hat{P}_w - P \|_1$ with $P$ being a random patch sampled from one of the K images and $\hat{P}_w$ being a patch rendered by the prior model conditioning on $w$ [S2]. $\mathcal {L}_\text {LPIPS}(\hat{P}_w, P)$ is a perceptual loss [S2]. The parameter $\lambda_{LPIPS}$ is set to 0.2 [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of optimization iterations and the cost of rendering and CLIP encoding per iteration. Not given in the sources.
*   **Memory Complexity:** Requires storing the latent code $w$, gradients, and intermediate rendered images. Not given in the sources.
*   **Strengths:** Allows for text-driven control of 3D scene attributes without requiring 3D annotations [S1].
*   **Limitations:** Suffers from the linked local attribute problem, limiting independent control of facial features [S1]. It is sub-optimal compared to methods that address this limitation [S1].
*   **When to use:** When text-driven manipulation is desired and the linked attribute problem is acceptable or mitigated by other means [S1].
*   **When not to use:** When independent control of local facial attributes is required [S1].

## Background
Originated in the paper "FaceCLIPNeRF: Text-driven 3D Face Manipulation using Deformable Neural Radiance Fields" [S1]. It is based on the concept of inversion, a gradient-based optimization technique used to project images back to the latent space of generative models [S1]. The method was developed to enable text-driven manipulation of 3D faces using deformable Neural Radiance Fields [S1].

## Sources
[S1] FaceCLIPNeRF: Text-driven 3D Face Manipulation using Deformable Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.00321, fetched 2026-10-29
[S2] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
