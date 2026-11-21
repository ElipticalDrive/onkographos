# Feed-Forward Hierarchical Gaussian Diffusion

Also called: HiGDiff, Hierarchical Gaussian Diffusion

## Overview
HiGDiff is a feed-forward reconstruction method for computed tomography (CT) that predicts a set of 3D Gaussian primitives from degraded projections without per-scene optimization [S1, unofficial]. It solves the problem of sparse-view, low-dose, or limited-angle CT reconstruction by using two explicit Gaussian banks (structure and detail) initialized from anatomical cues and refined via a sequential structure-to-detail diffusion hierarchy [S1, unofficial]. It differs from single-bank Gaussian reconstruction by maintaining separate banks for global attenuation geometry and residual detail, and from dense voxel diffusion by operating on explicit Gaussian primitives with a sequential hierarchy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Degraded projection data $Y \in \mathbb{R}^{N_r}$, where $N_r$ is the number of rays. Each ray $r$ has an observed count $Y_r$ and a mask $M_r \in \{0, 1\}$ indicating if the ray is observed.
*   **Output:** A set of 3D Gaussian primitives $\mathcal{G} = \mathcal{G}_s \cup \mathcal{G}_d$, where $\mathcal{G}_s$ is the structure bank and $\mathcal{G}_d$ is the detail bank. Each Gaussian $i$ in bank $b$ is defined by mean $m_{b,i}$, covariance $\Sigma_{b,i}$ (parameterized by scale and rotation), opacity $a_{b,i}$, and color/attenuation attributes. The final output is the reconstructed attenuation field rendered from these Gaussians.

**Data Structures:**
*   **Physics Features ($F_{phy}$):** A multi-scale feature pyramid extracted from the initial volume estimate.
*   **Anchors ($K$):** A set of landmark-free anatomical anchor points predicted from $F_{phy}$.
*   **Foreground Capacity Field ($F_{fg}$):** A continuous scalar field predicting the density/capacity for Gaussian placement.
*   **Gaussian Bank ($\mathcal{G}_b$):** A fixed-cardinality set of Gaussians. Each Gaussian $i$ has attributes $\xi_{b,i} = [m_{b,i}, \Sigma_{b,i}, a_{b,i}, \dots]$.
*   **Diffusion State ($\delta \xi_{b,i}^t$):** The noise-corrupted Gaussian attributes at diffusion timestep $t$.

**Parameters:**
*   $I_0$: Incident photon count.
*   $\bar{\alpha}_{b,t}$: Cumulative noise schedule coefficient for bank $b$ at step $t$.
*   $\eta$: DDIM inversion parameter (set to 0 for deterministic inversion).
*   $\lambda_{kp}, \lambda_{fg}, \lambda_b$: Loss weights for anchor consistency, capacity field, and diffusion loss respectively.
*   $w(q)$: Attention weighting function, $w(q) = 0.15 + 0.85q$.

**Steps:**

1.  **Initialization:**
    a. Compute estimated photon count for each ray: $I_r = I_0 \exp(-Y_r)$ [S1, unofficial].
    b. Compute normalized inverse-noise reliability $W_r$:
       $$ W_r = \frac{\frac{\max(I_r, 1)}{M_r}}{\max_{l:M_l=1} \max(I_l, 1)} $$
       where $M_r$ is the observation mask [S1, unofficial].
    c. Compute initial volume estimate $V_0$ using filtered back-projection (FBP): $V_0 = \text{FBP}_\theta(Y)$ [S1, unofficial].
    d. Compute physics features $F_{phy}$ using a physics encoder $E_{phy}$ on $V_0$ and the back-projected residual weighted by $W$:
       $$ F_{phy} = E_{phy}(V_0, \text{UBP}_\theta(W \bullet [Y - \text{Proj}_\theta(V_0)])) $$
       where $\text{UBP}$ is unweighted back-projection and $\text{Proj}$ is forward projection [S1, unofficial].
    e. Predict anatomical anchors $K$ and foreground capacity field $F_{fg}$:
       $$ K = E_{kp}(F_{phy}) $$
       $$ F_{fg} = E_{fg}(F_{phy}, K) $$
       [S1, unofficial].
    f. Initialize the structure bank $\mathcal{G}_{s,0}$ and detail bank $\mathcal{G}_{d,0}$ using $K$ and $F_{fg}$. Positions are initialized in normalized scanner coordinates $[-1, 1]^3$ [S1, unofficial].

2.  **Structure Gaussian Diffusion (Stage I):**
    For each diffusion step $t$ from $T$ down to 1:
    a. For each structure primitive $i$:
       i. Sample physics features at current mean $m_{s,i}^t$: $f_{s,i}^t = \text{Samp}(F_{phy}, m_{s,i}^t)$ [S1, unofficial].
       ii. Compute context $c_{s,i}^t$ by concatenating initial token $z_{s,i}$, sampled features $f_{s,i}^t$, and inverse-standardized initial attributes $\phi_s(g_{s,i,0})$, then passing through fusion MLP $M_s$ and attention weighting $w(q_{s,i}^t)$:
           $$ c_{s,i}^t = w(q_{s,i}^t) M_s [z_{s,i}, f_{s,i}^t, \phi_s(g_{s,i,0})] $$
           [S1, unofficial].
       iii. Predict noise $\hat{\epsilon}_{s,i}^t$ using structure denoiser $D_s$:
           $$ \hat{\epsilon}_{s,i}^t = D_s(\delta \xi_{s,i}^t, c_{s,i}^t, t) $$
           [S1, unofficial].
    b. Update state using DDIM inverse step ($\eta=0$):
       $$ \delta \xi_{s,i}^{t-1} = \frac{1}{\sqrt{\bar{\alpha}_{s,t}}} \left( \delta \xi_{s,i}^t - \sqrt{1 - \bar{\alpha}_{s,t}} \hat{\epsilon}_{s,i}^t \right) $$
       [S1, unofficial].
    c. Apply inverse standardization $\phi_s^{-1}$ and add correction to $\mathcal{G}_{s,0}$. Apply center clipping and positive numerical floors to ensure valid attributes (e.g., positive scales) [S1, unofficial].

3.  **Detail Gaussian Diffusion (Stage I):**
    a. Construct stopped conditions for detail diffusion using the refined structure bank. The detail bank is initialized based on the structure bank's output [S1, unofficial].
    b. For each diffusion step $t$ from $T$ down to 1:
       i. For each detail primitive $i$:
          - Predict noise $\hat{\epsilon}_{d,i}^t$ using detail denoiser $D_d$ conditioned on the structure bank and current detail state.
       ii. Update state using DDIM inverse step:
           $$ \delta \xi_{d,i}^{t-1} = \frac{1}{\sqrt{\bar{\alpha}_{d,t}}} \left( \delta \xi_{d,i}^t - \sqrt{1 - \bar{\alpha}_{d,t}} \hat{\epsilon}_{d,i}^t \right) $$
           [S1, unofficial].
    c. Apply inverse standardization $\phi_d^{-1}$ and domain validity checks (center clipping, positive floors) [S1, unofficial].

4.  **Refinement (Stage II):**
    After freezing Stage I modules, a refinement network is trained to correct residual errors after dense voxelization, using stop-gradient to preserve the division between structure and detail [S1, unofficial].

5.  **Rendering:**
    Render the attenuation field using the Gaussian renderer:
    $$ \mathcal{R}_G(\mathcal{G}_b)(x) = \sum_{i=1}^{N_b} a_{b,i} \mathcal{G}_G(x; m_{b,i}, \Sigma_{b,i}) $$
    where $\mathcal{G}_G(x; m, \Sigma) = \exp(-\frac{1}{2}(x-m)^\top \Sigma^{-1}(x-m))$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

def normalized_inverse_noise_reliability(Y, I0, M):
    """
    Y: (N_r,) observed projections
    I0: float, incident photon count
    M: (N_r,) binary mask
    """
    I_r = I0 * torch.exp(-Y)
    max_I_r = torch.clamp(I_r, min=1.0)
    numerator = max_I_r / M.clamp(min=1e-9) # Avoid div by zero, M is 0 or 1
    # Max over observed rays
    max_denom = torch.max(torch.where(M == 1, max_I_r, torch.tensor(0.0, device=Y.device)))
    W_r = numerator / max_denom
    return W_r

def initialize_banks(F_phy, K, F_fg, N_s, N_d):
    """
    Initialize structure and detail Gaussian banks from anchors and capacity field.
    Returns: G_s_0, G_d_0
    """
    # Implementation depends on specific initialization logic from E_fg and K
    # G_s_0: (N_s, 3) positions, (N_s, 3) scales, (N_s, 4) quaternions, (N_s, 1) opacities
    # G_d_0: (N_d, 3) positions, (N_d, 3) scales, (N_d, 4) quaternions, (N_d, 1) opacities
    # Placeholder for specific anchor-to-Gaussian mapping logic
    G_s_0 = init_structure_from_anchors(K, N_s)
    G_d_0 = init_detail_from_capacity(F_fg, N_d)
    return G_s_0, G_d_0

def structure_diffusion_step(G_s_t, F_phy, D_s, alpha_bar_t, phi_s, phi_s_inv):
    """
    One step of structure diffusion reverse process.
    """
    N_s = G_s_t.shape[0]
    m_s = G_s_t[:, :3]
    
    # Sample physics features at current means
    f_s = sample_features(F_phy, m_s) # (N_s, C)
    
    # Compute context
    # z_s is initial token, g_s_0 is initial attributes
    # This assumes z_s and g_s_0 are available or stored
    # c_s = w(q) * M_s([z_s, f_s, phi_s(g_s_0)])
    # q is attention weight, computed from current state
    q_s = compute_attention_weight(m_s)
    w_q = 0.15 + 0.85 * q_s
    
    # Concatenate and pass through MLP
    context = torch.cat([z_s, f_s, phi_s(G_s_t[:, 3:])], dim=-1)
    c_s = w_q.unsqueeze(-1) * M_s(context)
    
    # Predict noise
    delta_xi_t = G_s_t # Assuming G_s_t contains the noise-corrupted attributes
    eps_hat = D_s(delta_xi_t, c_s, t)
    
    # DDIM Inverse Step (eta=0)
    delta_xi_t_minus_1 = (1.0 / math.sqrt(alpha_bar_t)) * (delta_xi_t - math.sqrt(1 - alpha_bar_t) * eps_hat)
    
    # Inverse standardization and validity checks
    G_s_t_minus_1 = phi_s_inv(delta_xi_t_minus_1) + G_s_0 # Correction added to initialization
    G_s_t_minus_1 = center_clip(G_s_t_minus_1)
    G_s_t_minus_1 = positive_floor(G_s_t_minus_1)
    
    return G_s_t_minus_1

def detail_diffusion_step(G_d_t, G_s, F_phy, D_d, alpha_bar_t, phi_d, phi_d_inv):
    """
    One step of detail diffusion reverse process.
    """
    N_d = G_d_t.shape[0]
    
    # Condition on structure bank
    # Context includes structure bank features and current detail state
    context_d = compute_detail_context(G_s, G_d_t, F_phy)
    
    eps_hat = D_d(G_d_t, context_d, t)
    
    delta_xi_t_minus_1 = (1.0 / math.sqrt(alpha_bar_t)) * (G_d_t - math.sqrt(1 - alpha_bar_t) * eps_hat)
    
    G_d_t_minus_1 = phi_d_inv(delta_xi_t_minus_1) + G_d_0
    G_d_t_minus_1 = center_clip(G_d_t_minus_1)
    G_d_t_minus_1 = positive_floor(G_d_t_minus_1)
    
    return G_d_t_minus_1

def render_gaussians(G_s, G_d):
    """
    Render attenuation field from structure and detail banks.
    """
    # Sum of Gaussian contributions
    # R_G(G_b)(x) = sum(a_i * exp(-0.5 * (x-m_i)^T Sigma_i^-1 (x-m_i)))
    # This is typically done via rasterization in a renderer
    attenuation = render_bank(G_s) + render_bank(G_d)
    return attenuation

def train_stage_I(Y, N_s, N_d, D_s, D_d, E_phy, E_kp, E_fg, M_s, phi_s, phi_d):
    """
    Stage I: Alternating structure and detail updates.
    """
    W_r = normalized_inverse_noise_reliability(Y, I0, M)
    V_0 = FBP(Y)
    F_phy = E_phy(V_0, UBP(W_r * (Y - Proj(V_0))))
    
    K = E_kp(F_phy)
    F_fg = E_fg(F_phy, K)
    
    G_s_0, G_d_0 = initialize_banks(F_phy, K, F_fg, N_s, N_d)
    
    # Diffusion loop
    for t in T down to 1:
        # Structure update
        G_s_t = add_noise(G_s_0, t)
        G_s_t_minus_1 = structure_diffusion_step(G_s_t, F_phy, D_s, alpha_bar_s_t, phi_s, phi_s_inv)
        
        # Detail update (conditioned on structure)
        G_d_t = add_noise(G_d_0, t)
        G_d_t_minus_1 = detail_diffusion_step(G_d_t, G_s_t_minus_1, F_phy, D_d, alpha_bar_d_t, phi_d, phi_d_inv)
        
        # Compute losses
        loss_diff = compute_diffusion_loss(eps_hat, eps_true)
        loss_aux = compute_aux_loss(K, F_fg, G_s, G_d)
        
        # Backpropagate
        loss_diff.backward()
        loss_aux.backward()

def train_stage_II(G_s, G_d, Y, RefinementNet):
    """
    Stage II: Refinement network training.
    """
    # Freeze Stage I modules
    # Render Gaussians
    Attenuation = render_gaussians(G_s, G_d)
    
    # Voxelize and refine
    Voxel = voxelize(Attenuation)
    Refined_Voxel = RefinementNet(Voxel)
    
    # Loss against reference
    loss = L1(Refined_Voxel, Reference_CT)
    loss.backward()
```

## Operating characteristics
*   **Time Complexity:** Inference is feed-forward, consisting of a single pass through the physics encoder, anchor/capacity predictors, and two diffusion hierarchies. The complexity is dominated by the diffusion steps ($T$) and the size of the Gaussian banks ($N_s, N_d$). Each diffusion step involves sampling features and passing through MLPs/denoisers.
*   **Memory Complexity:** Requires storing the physics feature pyramid, the two Gaussian banks, and the diffusion states during inference. The fixed-cardinality banks make memory usage predictable.
*   **Performance:** Designed for extreme CT reconstruction (sparse-view, low-dose, limited-angle). It outperforms single-bank Gaussian reconstruction and dense voxel diffusion in these challenging settings by separating structure and detail [S1, unofficial].
*   **Strengths:** Handles degraded projections well; explicit Gaussian representation allows for efficient rendering; hierarchical diffusion improves detail recovery.
*   **Limitations:** Requires careful tuning of diffusion schedules and loss weights; initialization from anchors may fail if anatomical cues are absent; two-stage training adds complexity.

## Background
HiGDiff was proposed in "Feed-Forward Hierarchical Gaussian Diffusion for Extreme CT Reconstruction" (2026) [S1, unofficial]. It builds on the line of work using discretized Gaussian functions for continuous representation [S1, unofficial]. It extends DiffGS, which demonstrated generative modeling of Gaussian attributes, and UltraGS, which adapted Gaussian attributes to medical image formation. HiGDiff integrates explicit Gaussian representation, spatially selective capacity allocation, and a feed-forward structure-to-detail diffusion hierarchy [S1, unofficial].

## Sources
[S1] Feed-Forward Hierarchical Gaussian Diffusion for Extreme CT Reconstruction, https://doi.org/10.48550/arxiv.2608.15343, fetched 2026-10-26 (unofficial)
