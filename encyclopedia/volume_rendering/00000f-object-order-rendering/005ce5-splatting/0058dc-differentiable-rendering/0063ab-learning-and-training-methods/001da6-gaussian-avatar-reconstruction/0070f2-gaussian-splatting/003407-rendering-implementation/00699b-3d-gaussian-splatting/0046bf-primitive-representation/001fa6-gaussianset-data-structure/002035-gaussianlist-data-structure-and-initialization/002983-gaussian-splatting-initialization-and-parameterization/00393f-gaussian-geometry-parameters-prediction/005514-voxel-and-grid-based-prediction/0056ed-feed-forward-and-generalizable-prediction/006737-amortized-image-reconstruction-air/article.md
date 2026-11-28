# Amortized Image Reconstruction (AIR)
Also called: Self-Supervised Feed-Forward 2D Gaussian Splatting

## Overview
Amortized Image Reconstruction (AIR) is a self-supervised feed-forward framework that reconstructs a target image into a set of 2D Gaussian primitives in a single network pass, eliminating the costly per-image iterative optimization required by prior methods like GaussianImage [S1, unofficial]. Unlike Instant-GI, which relies on handcrafted priors and pseudo-labels for initialization, AIR uses a stage-wise residual architecture to progressively predict Gaussian primitives from reconstruction residuals, achieving higher quality with encoding times of 160–300 ms [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A target RGB image $I_{gt} \in \mathbb{R}^{H \times W \times 3}$ [S1, unofficial].
*   **Output:** A set of 2D Gaussian primitives $\mathcal{G} = \{\mu_n, \Sigma_n, c_n\}_{n=1}^N$, where $N$ is the number of primitives [S1, unofficial].

**Data Structures:**
Each Gaussian primitive $n$ is defined by:
1.  **Center:** $\mu_n \in \mathbb{R}^2$, the 2D coordinates on the image plane [S1, unofficial].
2.  **Covariance:** $\Sigma_n \in \mathbb{R}^{2 \times 2}$, a symmetric positive-definite matrix [S1, unofficial]. To ensure positive definiteness, it is parameterized by a rotation angle $\theta_n$ and scale factors $s_{n1}, s_{n2}$:
    $$ \Sigma_n = (R_n S_n) (R_n S_n)^T $$
    where $R_n$ is a 2D rotation matrix determined by $\theta_n$, and $S_n = \text{diag}(s_{n1}, s_{n2})$ [S1, unofficial].
3.  **Color:** $c_n \in \mathbb{R}^3$, the RGB color value [S1, unofficial]. The source notes $c_n$ absorbs opacity in the baseline GaussianImage formulation [S1, unofficial].

**Rendering:**
The rendered image $I_{render}$ is computed via differentiable rasterization $R(\cdot)$ [S1, unofficial]. For a pixel $p$, the color $C_p$ is a weighted aggregation of all Gaussians $n$ covering $p$:
$$ C_p = \frac{\sum_{n \in \mathcal{N}(p)} c_n \alpha_{n,p}}{\sum_{n \in \mathcal{N}(p)} \alpha_{n,p}} $$
where $\mathcal{N}(p)$ is the set of Gaussians covering pixel $p$, and the weight $\alpha_{n,p}$ is:
$$ \alpha_{n,p} = \exp \left( -\frac{1}{2} d_{n,p}^T \Sigma_n^{-1} d_{n,p} \right) $$
where $d_{n,p} = p - \mu_n$ is the offset vector [S1, unofficial].

**Inference Algorithm:**
AIR reconstructs an image through stage-wise residual Gaussian prediction [S1, unofficial].
1.  **Initialization:** Start with an empty Gaussian set $\mathcal{G}_0 = \emptyset$ and rendered image $I_0 = 0$ [S1, unofficial].
2.  **Stage-wise Prediction:** For each stage $k = 1, \dots, K$:
    a.  Compute the current reconstruction residual $R_{k-1} = I_{gt} - I_{k-1}$, where $I_{k-1} = R(\mathcal{G}_{k-1})$ [S1, unofficial].
    b.  The feed-forward predictor network takes $I_{gt}$ and $R_{k-1}$ as input to predict a new set of Gaussian primitives $\Delta \mathcal{G}_k$ [S1, unofficial].
    c.  **Stage Control:** An explicit mechanism activates new primitives only in under-reconstructed regions (where residual error is high) to ensure compactness [S1, unofficial].
    d.  Update the accumulated set: $\mathcal{G}_k = \mathcal{G}_{k-1} \cup \Delta \mathcal{G}_k$ [S1, unofficial].
    e.  Render the current accumulated set to get $I_k = R(\mathcal{G}_k)$ [S1, unofficial].
3.  **Final Output:** The final Gaussian set is $\mathcal{G}_K$ [S1, unofficial].

**Training Strategy (Predict–Optimize–Distill):**
Direct supervision with rendering loss is unstable due to the under-constrained mapping from image space to Gaussian space [S1, unofficial]. AIR uses a distillation approach:
1.  **Predict:** The predictor outputs $\Delta \mathcal{G}_k$ for the current stage [S1, unofficial].
2.  **Optimize:** A detached copy of the accumulated Gaussian set $\mathcal{G}_{k-1} \cup \Delta \mathcal{G}_k$ is refined using a few steps of differentiable rendering optimization (gradient descent on Gaussian parameters) to minimize $L_{render}(R(\mathcal{G}), I_{gt})$ [S1, unofficial]. Let the refined set be $\mathcal{G}^*_{k}$ [S1, unofficial].
3.  **Distill:** The refined set $\mathcal{G}^*_{k}$ serves as a regression target in Gaussian space [S1, unofficial]. The predictor is supervised by the difference between its output $\Delta \mathcal{G}_k$ and the increment required to reach $\mathcal{G}^*_{k}$ from $\mathcal{G}_{k-1}$ [S1, unofficial].
4.  **Finetuning:** After the POD training strategy stabilizes the multi-stage predictor, AIR is jointly finetuned across all stages with direct rendering supervision on the whole Gaussian set $\mathcal{G}_K$ [S1, unofficial].

**Quantization:**
An image-adaptive quantizer is applied to the predicted Gaussian parameters to enable compact Gaussian storage without per-image post-optimization [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
from typing import List, Dict, Tuple

def compute_covariance(theta: torch.Tensor, s: torch.Tensor) -> torch.Tensor:
    """Compute Sigma = (R*S)*(R*S)^T from angle theta and scale s."""
    cos_t = torch.cos(theta)
    sin_t = torch.sin(theta)
    R = torch.stack([torch.stack([cos_t, -sin_t]), torch.stack([sin_t, cos_t])])
    S = torch.diag_embed(s)
    RS = torch.matmul(R, S)
    return torch.matmul(RS, RS.transpose(-2, -1))

def render_gaussians(gaussians: List[Dict], img_h: int, img_w: int) -> torch.Tensor:
    """Differentiable rasterization of 2D Gaussians. Returns rendered image (H, W, 3)."""
    out_img = torch.zeros((img_h, img_w, 3), device=gaussians[0]['mu'].device)
    for y in range(img_h):
        for x in range(img_w):
            pixel_pos = torch.tensor([x, y], dtype=torch.float32, device=gaussians[0]['mu'].device)
            numerator = torch.zeros(3, device=gaussians[0]['mu'].device)
            denominator = torch.tensor(0.0, device=gaussians[0]['mu'].device)
            for g in gaussians:
                mu = g['mu']
                cov = g['cov']
                try:
                    cov_inv = torch.inverse(cov)
                except RuntimeError:
                    continue
                d = pixel_pos - mu
                alpha = torch.exp(-0.5 * torch.matmul(d.t(), torch.matmul(cov_inv, d)))
                if alpha < 1e-6:
                    continue
                numerator += g['c'] * alpha
                denominator += alpha
            if denominator > 0:
                out_img[y, x] = numerator / denominator
    return out_img

def optimize_gaussians(gaussians: List[Dict], target: torch.Tensor, steps: int) -> List[Dict]:
    """Refines a detached copy of Gaussians via gradient descent to match target image."""
    # [G] Standard Adam optimizer with L1 loss is assumed for this step.
    opt_gaussians = [dict(g) for g in gaussians]
    for _ in range(steps):
        rendered = render_gaussians(opt_gaussians, target.shape[0], target.shape[1])
        loss = torch.nn.functional.l1_loss(rendered, target)
        loss.backward()
        # [G] Optimizer step and zero_grad omitted for brevity
    return opt_gaussians

def compute_gaussian_space_loss(predicted_new: List[Dict], optimized_total: List[Dict], accumulated: List[Dict]) -> torch.Tensor:
    """Computes regression loss between predicted increments and optimized increments in Gaussian parameter space."""
    # [G] Standard L1/L2 loss between predicted parameters and optimized parameters is assumed.
    loss = torch.tensor(0.0, device=predicted_new[0]['mu'].device)
    for p, o in zip(predicted_new, optimized_total[len(accumulated):]):
        loss += torch.nn.functional.l1_loss(p['mu'], o['mu'])
        loss += torch.nn.functional.l1_loss(p['theta'], o['theta'])
        loss += torch.nn.functional.l1_loss(p['s'], o['s'])
        loss += torch.nn.functional.l1_loss(p['c'], o['c'])
    return loss

def stage_control_mask(residual: torch.Tensor, threshold: float) -> torch.Tensor:
    """Returns binary mask activating primitives only in under-reconstructed regions."""
    # [G] Standard thresholding on residual magnitude is assumed.
    return (residual.abs().mean(dim=-1) > threshold).float()

class AIRPredictor(nn.Module):
    """Feed-forward network predicting Gaussian primitives."""
    def __init__(self):
        super().__init__()
        # [G] Encoder-Decoder architecture details omitted as not specified in source
        self.encoder = nn.Identity() 
        self.decoder = nn.Identity()

    def forward(self, image: torch.Tensor, residual: torch.Tensor) -> List[Dict]:
        """Predicts a list of Gaussian primitives given the input image and residual."""
        # Implementation specific to the paper's architecture
        # Inputs: image (H, W, 3), residual (H, W, 3)
        # Output: List of dicts with keys 'mu', 'theta', 's', 'c'
        pass # [G] Network forward pass implementation omitted

def train_step_aire(image_gt: torch.Tensor, predictor: AIRPredictor, num_stages: int = 3, threshold: float = 0.05):
    """Simplified training loop illustrating the POD strategy."""
    gaussians = []
    current_rendered = torch.zeros_like(image_gt)
    
    for k in range(num_stages):
        residual = image_gt - current_rendered
        
        # Predict new primitives based on residual
        new_primitives = predictor(image_gt, residual)
        
        # Stage Control: filter predictions based on residual magnitude
        mask = stage_control_mask(residual, threshold)
        # [G] Masking logic applied to predicted primitives omitted
        
        # Optimize (Distillation Target Generation)
        temp_gaussians = gaussians + new_primitives
        refined_gaussians = optimize_gaussians(temp_gaussians, image_gt, steps=5)
        
        # Distill
        distill_loss = compute_gaussian_space_loss(new_primitives, refined_gaussians, gaussians)
        distill_loss.backward()
        
        # Update accumulated set
        gaussians = refined_gaussians
        current_rendered = render_gaussians(gaussians, image_gt.shape[0], image_gt.shape[1])
        
    return gaussians
```

**Parameters:**
*   **Number of Stages ($K$):** Not given in the sources, but described as "stage-wise" [S1, unofficial].
*   **Encoding Time:** 160–300 ms on Kodak and DIV2K datasets [S1, unofficial].
*   **Optimization Steps (POD):** "A few" differentiable rendering optimization steps are used during the distillation phase [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Under-constrained Mapping:** Direct supervision with rendering loss leads to unstable optimization [S1, unofficial]. The POD strategy is required to stabilize training [S1, unofficial].
*   **Primitive Allocation:** Without Stage Control, predicting primitives uniformly may lead to redundancy [S1, unofficial]. The explicit Stage Control mechanism ensures new primitives are only activated in under-reconstructed regions [S1, unofficial].
*   **Compactness:** The goal is to find the smallest Gaussian set $G^*$ satisfying a fidelity target $\epsilon$: $G^* \in \arg \min_G |G| \text{ s.t. } L_{render}(R(G), I_{gt}) \le \epsilon$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Encoding time is 160–300 ms per image on standard datasets (Kodak, DIV2K), significantly faster than iterative methods like GaussianImage which require per-image optimization [S1, unofficial].
*   **Memory Complexity:** Uses an image-adaptive quantizer for compact Gaussian storage, reducing memory footprint compared to unquantized representations [S1, unofficial].
*   **Strengths:** Fully self-supervised and end-to-end; eliminates per-image test-time optimization; achieves better reconstruction quality than Instant-GI and GaussianImage; no reliance on handcrafted priors (e.g., PPM, dithering) [S1, unofficial].
*   **Limitations:** Relies on a trained feed-forward network, so it generalizes based on the training data distribution; the POD training strategy adds complexity to the training pipeline compared to direct supervised learning [S1, unofficial].
*   **When to Use:** When fast, high-quality image reconstruction into 2D Gaussian splats is required without per-image optimization overhead [S1, unofficial].
*   **When Not to Use:** When per-image iterative refinement is acceptable for higher fidelity, or when the input images are significantly out-of-distribution from the training set (Kodak/DIV2K) [S1, unofficial].

## Background
AIR was introduced by Yuesong Wang in a preprint titled "AIR: Amortized Image Reconstruction Framework for Self-Supervised Feed-Forward 2D Gaussian Splatting" [S1, unofficial]. It addresses the limitations of existing 2D Gaussian image reconstruction methods, which either rely on costly per-image iterative optimization (GaussianImage, Image-GS) or handcrafted priors and pseudo-labels (Instant-GI) [S1, unofficial]. The method builds on the 2D Gaussian splatting formulation introduced in GaussianImage [S1, unofficial].

## Sources
[S1] AIR: Amortized Image Reconstruction Framework for Self-Supervised Feed-Forward 2D Gaussian Splatting, https://doi.org/10.48550/arXiv.2605.20820, fetched 2026-10-29 (unofficial)
