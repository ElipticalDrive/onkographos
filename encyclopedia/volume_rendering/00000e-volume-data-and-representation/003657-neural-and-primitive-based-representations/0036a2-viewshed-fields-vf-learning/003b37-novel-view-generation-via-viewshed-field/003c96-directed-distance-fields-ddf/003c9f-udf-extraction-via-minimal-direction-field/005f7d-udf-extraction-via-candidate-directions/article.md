# UDF Extraction via Candidate Directions

Also called: UDF depth estimation from candidate directions, v* Extraction via Candidate Directions

## Overview
This method estimates the Unoriented Distance Field (UDF) value and the optimal direction $v^*$ at a query point $p$ by using a small SIREN network to generate $K_c$ candidate directions, computing associated distances and visibility, and aggregating them with softmax weights [S1]. Unlike computing $v^*$ via the gradient of the UDF (backward pass), which is noisier, this fitting approach provides a stable direction estimate by explicitly optimizing a set of candidate directions [S1].

## How it works
**Inputs:**
*   A trained (P)DDF network providing distance $d(p, v)$ and visibility $\xi(p, v)$ for any direction $v$ at point $p$ [S1].
*   A bounding volume $B$ for sampling training points [S1].
*   Hyperparameters: $K_c=5$, $\tau_n = 5 \times 10^{-3}$, $\tau_d = 0.1$, $\eta_T$, $\varepsilon_s$ (from explicit composition) [S1].

**Outputs:**
*   $\widehat{\text{UDF}}(p)$: The estimated unoriented distance [S1].
*   $v^*(p)$: The estimated optimal direction [S1].

**Data Structures:**
*   Network $g_v$: A SIREN network mapping $\mathbb{R}^3 \to S^2 \times K_c$ [S1]. It consists of 5 hidden layers, each with 128 units [S1]. The output is reshaped to $K_c$ vectors in $\mathbb{R}^2$, which are normalized to unit length to form directions in $S^2$ [G].

**Steps:**
1.  **Candidate Generation:** For a query point $p$, compute $K_c$ candidate directions $\{v_i^*(p)\}_{i=1}^{K_c}$ by passing $p$ through $g_v(p)$ and normalizing the resulting vectors [S1].
2.  **Distance and Visibility:** For each candidate $v_i^*(p)$, query the trained (P)DDF to obtain the distance $d(p, v_i^*(p))$ and visibility $\xi(p, v_i^*(p))$ [S1].
3.  **Weight Calculation:** Compute softmax weights $\omega_{\zeta_v^{(i)}}(p)$ based on visibility and distance:
    $$\omega_{\zeta_v^{(i)}}(p) = \text{Softmax}\left(\left\{\frac{\eta_T^{-1}\xi(p, v_i^*(p))}{\varepsilon_s + d(p, v_i^*(p))}\right\}_i\right)$$
    where $\eta_T$ and $\varepsilon_s$ are parameters from the explicit composition method [S1].
4.  **UDF Estimation:** Compute the UDF estimate as the weighted sum of distances:
    $$\widehat{\text{UDF}}(p) = \sum_i \omega_{\zeta_v^{(i)}}(p) d(p, v_i^*(p))$$ [S1].
5.  **Direction Estimation:** Compute the weighted average of candidate directions and normalize:
    $$\bar{v}^*(p) = \sum_i \omega_{\zeta_{v^*}}(p) v_i^*(p)$$
    $$v^*(p) = \frac{\bar{v}^*(p)}{\|\bar{v}^*(p)\|_2}$$ [S1].

**Training:**
To train $g_v$, minimize the loss $\mathcal{L}_{v^*}$ consisting of three terms [S1]:
1.  **Depth and Visibility:** Encourages minimum visible depth.
    $$\frac{1}{K_c} \sum_i d(p, v_i^*(p)) - \xi(p, v_i^*(p))$$
    The $d$ term linearly penalizes longer distances, and the $-\xi$ term pushes visibility to be high [S1].
2.  **Candidate Collapse Prevention:** Prevents candidates from collapsing to a single direction (local minima) [S1].
    $$\frac{2\tau_n}{K_c^2 - K_c} \sum_i \sum_{j \neq i} v_i^*(p)^T v_j^*(p)$$
3.  **Normal Alignment:** Encodes alignment with local surface normals $n(p, v_i^*(p))$ [S1].
    $$\frac{\tau_d}{K_c} \sum_i [v_i^*(p)^T n(p, v_i^*(p)) + 1]^2$$

**Parameters:**
*   $K_c = 5$ [S1].
*   $\tau_n = 5 \times 10^{-3}$ [S1].
*   $\tau_d = 0.1$ [S1].
*   Optimizer: Adam, LR $10^{-4}$, $\beta_1 = 0.9$, $\beta_2 = 0.999$ [S1].
*   Iterations: $10^4$ [S1].
*   Batch size: 4096 points uniformly drawn from $B$ ($p \sim U[B]$) [S1].

**Pitfalls:**
*   Computing $v^*$ via the backward pass (gradient of UDF) is noisier in practice than this candidate fitting approach [S1].
*   The method assumes $v^*(p)$ is not multivalued (i.e., $p$ is not on the surface $S$ or the medial surface) for the normal alignment term to be well-defined [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class CandidateDirectionNetwork(nn.Module):
    def __init__(self, Kc=5):
        super().__init__()
        self.Kc = Kc
        # 5 hidden layers, 128 units each, mapping R^3 to R^(2*Kc) [S1]
        self.net = nn.Sequential(
            nn.Linear(3, 128), nn.Sine(),
            nn.Linear(128, 128), nn.Sine(),
            nn.Linear(128, 128), nn.Sine(),
            nn.Linear(128, 128), nn.Sine(),
            nn.Linear(128, 128), nn.Sine(),
            nn.Linear(128, 2 * Kc)
        )

    def forward(self, p):
        # p: (B, 3)
        out = self.net(p)  # (B, 2*Kc)
        # Reshape to (B, Kc, 2) and normalize to S^2 [G]
        dirs = out.view(-1, self.Kc, 2)
        dirs = F.normalize(dirs, dim=-1)
        return dirs

def compute_udf_and_vstar(p, candidate_net, eta_T, eps_s, tau_n, tau_d, Kc=5):
    # 1. Generate candidates [S1]
    v_candidates = candidate_net(p)  # (B, Kc, 2)

    # 2. Compute distance and visibility for each candidate [S1]
    # External trained (P)DDF query for distance d and visibility xi
    d, xi = torch.zeros_like(v_candidates[:, :, 0]), torch.ones_like(v_candidates[:, :, 0])

    # 3. Compute weights [S1]
    # omega = Softmax( (eta_T^-1 * xi) / (eps_s + d) )
    logits = (xi / eta_T) / (eps_s + d)
    weights = F.softmax(logits, dim=1)  # (B, Kc)

    # 4. Compute UDF estimate [S1]
    udf_estimate = torch.sum(weights * d, dim=1)  # (B,)

    # 5. Compute v* estimate [S1]
    v_bar = torch.sum(weights.unsqueeze(-1) * v_candidates, dim=1)  # (B, 2)
    v_star = F.normalize(v_bar, dim=-1)  # (B, 2)

    return udf_estimate, v_star, weights, v_candidates, d, xi

def compute_training_loss(p, candidate_net, eta_T, eps_s, tau_n, tau_d, Kc=5):
    udf, v_star, weights, v_candidates, d, xi = compute_udf_and_vstar(
        p, candidate_net, eta_T, eps_s, tau_n, tau_d, Kc
    )

    # Term 1: Depth and Visibility [S1]
    term1 = torch.mean(torch.sum(d - xi, dim=1) / Kc)

    # Term 2: Candidate Collapse Prevention [S1]
    dot_products = torch.bmm(v_candidates, v_candidates.transpose(1, 2))  # (B, Kc, Kc)
    mask = 1.0 - torch.eye(Kc, device=p.device).unsqueeze(0)  # (1, Kc, Kc)
    dot_products = dot_products * mask
    term2 = (2 * tau_n / (Kc**2 - Kc)) * torch.mean(torch.sum(dot_products, dim=(1, 2)))

    # Term 3: Normal Alignment [S1]
    # Query surface normals n(p, v) from DDF
    normals = torch.zeros_like(v_candidates)
    vn_dot = torch.sum(v_candidates * normals, dim=-1)  # (B, Kc)
    term3 = (tau_d / Kc) * torch.mean(torch.sum((vn_dot + 1)**2, dim=1))

    loss = term1 + term2 + term3
    return loss
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of candidates $K_c$ and the cost of querying the (P)DDF. Each query point requires $K_c$ forward passes through the (P)DDF [S1].
*   **Memory Complexity:** Low, as it only stores the small SIREN network $g_v$ (5 layers, 128 units) and the batch of candidate directions [S1].
*   **Strengths:** Provides a stable and less noisy estimate of $v^*$ compared to gradient-based methods [S1]. Explicitly handles visibility and distance trade-offs via softmax weighting [S1].
*   **Limitations:** Requires training an additional network $g_v$ on top of the (P)DDF [S1]. Performance depends on the quality of the (P)DDF and the choice of hyperparameters ($K_c$, $\tau_n$, $\tau_d$) [S1].
*   **When to use:** When a robust UDF estimate and direction field are needed from a trained (P)DDF, especially in regions where gradient-based estimation is unstable [S1].

## Background
This method was introduced in the paper "Representing 3D Shapes with Probabilistic Directed Distance Fields" [S1]. The approach was developed to address the noise issues associated with computing $v^*$ via the backward pass of the UDF [S1].

## Sources
[S1] Representing 3D Shapes with Probabilistic Directed Distance Fields, https://doi.org/10.1109/cvpr52688.2022.01874, fetched 2026-10-26
