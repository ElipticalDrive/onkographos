# Face Reconstruction via Latent Code Optimization

Also called: Face Inversion, Inversion (Latent Code Optimization)

## Overview
This method solves an optimization problem to find a latent code that produces a good starting point for face geometry reconstruction, given $K$ views of a target identity [S1]. It recovers the latent code by minimizing photometric and perceptual losses on image patches rendered by a frozen prior volumetric model [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** $K$ views of a target identity (images $I_1, \dots, I_K$); a frozen prior volumetric model $\mathcal{M}$ trained on a dataset of identities [S1].
- **Outputs:** An optimized latent code $w_{\text{target}}$ [S1].

**Data Structures:**
- **Latent Code $w$:** A vector in the latent space of the prior model. The dimensionality is determined by the prior model's architecture (Not given in the sources).
- **Image Patch $P$:** A random patch sampled from one of the $K$ target images [S1].
- **Rendered Patch $\hat{P}_w$:** A patch rendered by the prior model $\mathcal{M}$ conditioned on latent code $w$ [S1].

**Algorithm Steps:**
1. **Initialization (Optional):** If using Nearest/Furthest Neighbour initialization, compute embeddings for target images and training identities, then select the best match to initialize $w$ [S1]. Otherwise, initialize $w$ randomly or with a mean prior code [G].
2. **Optimization Loop:** Minimize the objective function with respect to $w$ while keeping the prior model weights frozen [S1].
3. **Loss Calculation:** For each optimization step, sample a random patch $P$ from one of the $K$ target images [S1]. Render the corresponding patch $\hat{P}_w$ using the prior model with current $w$ [S1]. Compute $\mathcal{L}_{\text{recon}}$ and $\mathcal{L}_{\text{LPIPS}}$ [S1].
4. **Update:** Update $w$ using gradient descent [G].

**Formulas:**
The latent code $w_{\text{target}}$ is recovered by minimizing:
$$
w_{\text{target}} = \argmin_{w} \mathcal{L}_{\text{recon}} + \lambda_{\text{LPIPS}} \mathcal{L}_{\text{LPIPS}}
$$
where:
- $\mathcal{L}_{\text{recon}} = \| \hat{P}_w - P \|_1$ is the $L_1$ loss between the rendered patch and the target patch [S1].
- $\mathcal{L}_{\text{LPIPS}}(\hat{P}_w, P)$ is the Learned Perceptual Image Patch Similarity (LPIPS) perceptual loss [S1].
- $\lambda_{\text{LPIPS}} = 0.2$ is the weighting factor for the perceptual loss [S1].

**Parameters:**
- $\lambda_{\text{LPIPS}} = 0.2$ [S1].
- Optimization resolution: Same as the prior model's training resolution, after removing the background [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import torch

def optimize_latent_code(target_images, prior_model, lambda_lpips=0.2, num_iterations=1000):
    # Initialize w randomly
    w = torch.nn.Parameter(torch.randn(latent_dim))
    optimizer = torch.optim.Adam([w], lr=0.01)  # [G] standard optimizer choice
    for _ in range(num_iterations):
        optimizer.zero_grad()
        # Sample a random patch P from one of the K target images
        target_img = random.choice(target_images)
        h, w_img = target_img.shape[1], target_img.shape[2]
        patch_h, patch_w = 64, 64
        y, x = random.randint(0, h - patch_h), random.randint(0, w_img - patch_w)
        P = target_img[y:y+patch_h, x:x+patch_w]
        # Render full image with current w, then extract patch
        full_render = prior_model.render(w)
        P_hat_w = full_render[y:y+patch_h, x:x+patch_w]
        # Compute losses
        loss_recon = torch.mean(torch.abs(P_hat_w - P))
        loss_lpips = lpips_loss(P_hat_w, P)
        loss = loss_recon + lambda_lpips * loss_lpips
        loss.backward()
        optimizer.step()
    return w
```

**Edge Cases and Pitfalls:**
- **Background:** The optimization is performed after removing the background [S1]. Failure to do so may lead to incorrect alignment [G].
- **Resolution:** Optimization occurs at the prior model's resolution [S1]. Optimizing at a different resolution may yield suboptimal results [G].
- **Camera Calibration:** The method is sensitive to correct camera calibration, particularly for thin structures like eyes and eyelids [S1].

## Subtopics

### Nearest/Furthest Neighbour Initialisation
A method to initialize the latent code $w$ before optimization. It computes image embeddings for all target images using a pretrained face recognition network [S1]. It computes embeddings for frontal renderings of all reconstructed training identities. It calculates the mean embedding of the target image embeddings. It computes similarity between the target mean embedding and each training identity's frontal rendering embedding. It selects the training identity with the highest similarity (Nearest) or lowest similarity (Furthest) to initialize the target's latent code or model parameters [S1]. This provides a better starting point than random initialization, improving reconstruction quality [S1].

## Operating characteristics
- **Time Complexity:** Depends on the number of optimization iterations and the cost of rendering patches through the prior model. Not given in the sources.
- **Memory Complexity:** Requires storing the latent code $w$ and intermediate gradients. Not given in the sources.
- **Measured Performance:** The method produces good starting points for face geometry reconstruction, as shown in Table 2 of [S1]. The supplementary material shows visual examples for different initialization techniques in Table 3 and Fig. 21 [S1].
- **Strengths:** Provides a robust initialization for few-shot reconstruction. Uses perceptual loss to ensure high-frequency details are preserved [S1].
- **Limitations:** Sensitive to correct camera calibration [S1]. The prior model is trained on neutral faces with closed mouths, so it may fail for strong expressions and teeth [S1]. Performance does not strongly differ with the number of views [S1].

## Background
This method is part of the "Inversion" step in the pipeline described in [S1]. It is used to find a latent code that produces a good starting point for face geometry reconstruction, given $K$ views of a target identity [S1]. The prior model is trained on low resolution and is optimized to reconstruct a large set of identities from many views for each identity [S1]. After model fitting, the model should generate high-resolution novel views with intricate details like individual hair strands for a single target identity given as few as two views [S1].

## Sources
[S1] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
