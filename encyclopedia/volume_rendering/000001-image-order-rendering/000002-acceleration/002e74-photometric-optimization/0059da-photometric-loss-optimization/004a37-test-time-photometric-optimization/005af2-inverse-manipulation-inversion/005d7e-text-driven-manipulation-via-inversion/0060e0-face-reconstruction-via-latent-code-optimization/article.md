# Face Reconstruction via Latent Code Optimization

Also called: Face Inversion, Inversion (Latent Code Optimization)

## Overview
This method solves the initialization problem for few-shot face reconstruction by finding a latent code $w$ that produces a rendered image closely matching a target identity's input views. It sets apart from its parent [Text-driven Manipulation via Inversion](17409-text-driven-manipulation-via-inversion.md) by using photometric and perceptual losses on image patches rather than CLIP text embeddings, and from [Photometric Loss Optimization](17408-photometric-loss-optimization.md) by focusing specifically on recovering a latent code for a prior volumetric model rather than optimizing network weights directly. It provides a high-quality starting point for subsequent model fitting.

## How it works
**Inputs and Outputs:**
- **Inputs:** $K$ views of a target identity (images $I_1, \dots, I_K$); a frozen prior volumetric model $\mathcal{M}$ trained on a dataset of identities; a pretrained face recognition network for initialization variants.
- **Outputs:** An optimized latent code $w_{\text{target}}$.

**Data Structures:**
- **Latent Code $w$:** A vector in the latent space of the prior model. The dimensionality is determined by the prior model's architecture (Not given in the sources).
- **Image Patch $P$:** A random patch sampled from one of the $K$ target images.
- **Rendered Patch $\hat{P}_w$:** A patch rendered by the prior model $\mathcal{M}$ conditioned on latent code $w$.

**Algorithm Steps:**
1. **Initialization (Optional):** If using Nearest/Furthest Neighbour initialization, compute embeddings for target images and training identities, then select the best match to initialize $w$. Otherwise, initialize $w$ randomly or with a mean prior code.
2. **Optimization Loop:** Minimize the objective function with respect to $w$ while keeping the prior model weights frozen.
3. **Loss Calculation:** For each optimization step, sample a random patch $P$ from one of the $K$ target images. Render the corresponding patch $\hat{P}_w$ using the prior model with current $w$. Compute $\mathcal{L}_{\text{recon}}$ and $\mathcal{L}_{\text{LPIPS}}$.
4. **Update:** Update $w$ using gradient descent.

**Formulas:**
The latent code $w_{\text{target}}$ is recovered by minimizing:
$$
w_{\text{target}} = \argmin_{w} \mathcal{L}_{\text{recon}} + \lambda_{\text{LPIPS}} \mathcal{L}_{\text{LPIPS}}
$$
where:
- $\mathcal{L}_{\text{recon}} = \| \hat{P}_w - P \|_1$ is the $L_1$ loss between the rendered patch and the target patch.
- $\mathcal{L}_{\text{LPIPS}}(\hat{P}_w, P)$ is the Learned Perceptual Image Patch Similarity (LPIPS) perceptual loss.
- $\lambda_{\text{LPIPS}} = 0.2$ is the weighting factor for the perceptual loss [S1].

**Parameters:**
- $\lambda_{\text{LPIPS}} = 0.2$ [S1].
- Optimization resolution: Same as the prior model's training resolution, after removing the background [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_latent_code(target_images, prior_model, lambda_lpips=0.2, num_iterations=1000):
    """
    Optimizes a latent code w to match target images using a frozen prior model.
    
    Args:
        target_images: List of K images of the target identity.
        prior_model: Frozen volumetric prior model.
        lambda_lpips: Weight for LPIPS loss.
        num_iterations: Number of optimization steps.
        
    Returns:
        w_target: Optimized latent code.
    """
    # Initialize w (e.g., random or from Nearest Neighbour)
    w = initialize_w(target_images, prior_model)
    
    # Use an optimizer (e.g., Adam) for w
    optimizer = torch.optim.Adam([w], lr=0.01)
    
    for i in range(num_iterations):
        optimizer.zero_grad()
        
        # Sample a random target image and a random patch from it
        target_img = random.choice(target_images)
        P = sample_random_patch(target_img)
        
        # Render a patch using the prior model with current w
        # Note: The prior model must support rendering patches given w and camera params
        # Assuming camera params are fixed or sampled appropriately for the patch location
        # The source specifies optimizing at the same resolution as the prior model
        # after removing background. We assume the rendering function handles this.
        P_hat_w = prior_model.render_patch(w, patch_location=get_patch_location(P))
        
        # Compute L1 reconstruction loss
        loss_recon = torch.mean(torch.abs(P_hat_w - P))
        
        # Compute LPIPS perceptual loss
        # LPIPS typically requires pre-trained VGG weights
        loss_lpips = lpips_loss(P_hat_w, P)
        
        # Total loss
        loss = loss_recon + lambda_lpips * loss_lpips
        
        # Backpropagate and update w
        loss.backward()
        optimizer.step()
        
    return w
```

**Edge Cases and Pitfalls:**
- **Background:** The optimization is performed after removing the background [S1]. Failure to do so may lead to incorrect alignment.
- **Resolution:** Optimization occurs at the prior model's resolution [S1]. Optimizing at a different resolution may yield suboptimal results.
- **Camera Calibration:** The method is sensitive to correct camera calibration, particularly for thin structures like eyes and eyelids [S1].

## Subtopics

### Nearest/Furthest Neighbour Initialisation
A method to initialize the latent code $w$ before optimization. It computes image embeddings for all target images using a pretrained face recognition network [S1]. It computes embeddings for frontal renderings of all reconstructed training identities. It calculates the mean embedding of the target image embeddings. It computes similarity between the target mean embedding and each training identity's frontal rendering embedding. It selects the training identity with the highest similarity (Nearest) or lowest similarity (Furthest) to initialize the target's latent code or model parameters [S1]. This provides a better starting point than random initialization, improving reconstruction quality [S1].

## Operating characteristics
- **Time Complexity:** Depends on the number of optimization iterations and the cost of rendering patches through the prior model. Not given in the sources.
- **Memory Complexity:** Requires storing the latent code $w$ and intermediate gradients. Not given in the sources.
- **Measured Performance:** The method produces good starting points for face geometry reconstruction, as shown in Table 2 of [S1]. The supplementary material shows visual examples for different initialization techniques in Table 3 and Fig. 21 [S1].
- **Strengths:** Provides a robust initialization for few-shot reconstruction. Uses perceptual loss to ensure high-frequency details are preserved.
- **Limitations:** Sensitive to correct camera calibration [S1]. The prior model is trained on neutral faces with closed mouths, so it may fail for strong expressions and teeth [S1]. Performance does not strongly differ with the number of views [S1].

## Background
This method is part of the "Inversion" step in the pipeline described in [S1]. It is used to find a latent code that produces a good starting point for face geometry reconstruction, given $K$ views of a target identity [S1]. The prior model is trained on low resolution and is optimized to reconstruct a large set of identities from many views for each identity [S1]. After model fitting, the model should generate high-resolution novel views with intricate details like individual hair strands for a single target identity given as few as two views [S1].

## Sources
[S1] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
