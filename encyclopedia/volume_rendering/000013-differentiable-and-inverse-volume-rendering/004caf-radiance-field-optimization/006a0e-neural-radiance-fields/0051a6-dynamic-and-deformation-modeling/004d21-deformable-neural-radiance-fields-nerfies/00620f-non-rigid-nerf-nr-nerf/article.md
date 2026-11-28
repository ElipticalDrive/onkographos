# Non-rigid NeRF (NR-NeRF)

Also called: NR-NeRF

## Overview
Non-rigid NeRF (NR-NeRF) is a method for modeling time-varying scene appearance by decomposing the dynamic scene into a static canonical volume and a learned deformation field [S1, S2, unofficial]. It implements scene deformation as a per-frame ray bending operator, allowing the rendering of novel views from monocular input sequences without requiring additional supervisory cues such as depth maps or scene flows [S1, unofficial]. It distinguishes itself from sibling methods like Nerfies by explicitly modeling a per-scene rigidity flag to constrain rigid regions, thereby preventing unwanted deformation in static parts of the scene [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of images $I_t$ captured at times $t$, along with their corresponding camera poses $P_t$.
*   **Outputs:** A rendered image for a target time $t$ and camera pose $P_{target}$.

**Data Structures:**
1.  **Canonical Neural Radiance Field ($\mathcal{N}_{c}$):** An MLP that maps a 3D position $\mathbf{x} \in \mathbb{R}^3$ to color $\mathbf{c}$ and volume density $\sigma$.
2.  **Ray Bending Network ($\mathcal{N}_{d}$):** An MLP that maps a canonical position $\mathbf{x}$ and a time embedding (or latent code) $t$ to a displacement vector $\Delta \mathbf{x}$.
3.  **Rigidity Network ($\mathcal{N}_{r}$):** An MLP that maps a canonical position $\mathbf{x}$ to a rigidity score $r \in [0, 1]$.

**Numbered Steps:**
1.  **Ray Sampling:** For a target camera pose and time $t$, sample points $\mathbf{x}_i$ along the ray in the canonical space.
2.  **Deformation (Ray Bending):** For each point $\mathbf{x}_i$, compute the displacement $\Delta \mathbf{x}_i$ using the Ray Bending Network:
    $$ \Delta \mathbf{x}_i = \mathcal{N}_{d}(\mathbf{x}_i, t) $$
    The observed position $\mathbf{x}'_i$ in the canonical space is derived by bending the ray. Note: Sources describe this as "ray bending" where the deformation field applies transformations [S2, unofficial]. The exact mapping from observation to canonical space involves solving for the canonical point that maps to the observed point, or vice versa. NR-NeRF models the deformation as a displacement from the canonical space to the observed space (or vice versa) [S1, unofficial]. Let us define the deformation as mapping canonical $\mathbf{x}$ to observed $\mathbf{x}'$:
    $$ \mathbf{x}' = \mathbf{x} + \Delta \mathbf{x} $$
    However, for volume rendering, we typically need the canonical coordinate corresponding to an observed ray. The method regularizes observed deformations [S1, unofficial].
3.  **Rigidity Masking:** Compute the rigidity score $r_i$ for each point:
    $$ r_i = \mathcal{N}_{r}(\mathbf{x}_i) $$
    The deformation is applied only to non-rigid parts. The effective displacement $\Delta \mathbf{x}^{eff}_i$ is modulated by the rigidity score:
    $$ \Delta \mathbf{x}^{eff}_i = (1 - r_i) \cdot \Delta \mathbf{x}_i $$
    This ensures rigid regions ($r_i \approx 1$) remain unchanged, while non-rigid regions ($r_i \approx 0$) are deformed [S2, unofficial].
4.  **Regularization:** Apply a divergence operator to the deformation field to impose a volume-preserving constraint. This stabilizes occluded areas with respect to monocular input views [S1, unofficial]. The divergence regularization term $\mathcal{L}_{div}$ is added to the loss function:
    $$ \mathcal{L}_{div} = \sum_i \left( \nabla \cdot \Delta \mathbf{x}^{eff}_i \right)^2 $$
5.  **Rendering:** Use the canonical radiance field $\mathcal{N}_{c}$ to query color and density at the deformed positions (or canonical positions if the ray bending is handled via inverse mapping). Standard volume rendering integration is performed along the ray.
6.  **Joint Training:** The rigidity network, ray bending network, and canonical neural radiance field are jointly trained [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class CanonicalNeRF(nn.Module):
    def __init__(self):
        super().__init__()
        # MLP mapping (x) -> (sigma, color)
        self.mlp = nn.Sequential(...)

    def forward(self, x):
        return self.mlp(x)

class RayBendingNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # MLP mapping (x, t) -> delta_x
        self.mlp = nn.Sequential(...)

    def forward(self, x, t):
        return self.mlp(torch.cat([x, t], dim=-1))

class RigidityNetwork(nn.Module):
    def __init__(self):
        super().__init__()
        # MLP mapping (x) -> rigidity_score
        self.mlp = nn.Sequential(...)

    def forward(self, x):
        return torch.sigmoid(self.mlp(x))

def compute_divergence(delta_x, x, eps=1e-5):
    """
    Approximate divergence of the deformation field delta_x with respect to x.
    Uses finite differences or automatic differentiation.
    """
    # Using autograd for exact divergence
    div = 0.0
    for i in range(3):
        # Compute partial derivative of delta_x[i] w.r.t x[i]
        d_dx = torch.autograd.grad(
            delta_x[:, i].sum(), x, create_graph=True
        )[0]
        div += d_dx
    return div

def render_ray(ray_origin, ray_direction, t, canonical_nerf, bending_net, rigidity_net, num_samples):
    """
    Renders a single ray using NR-NeRF.
    """
    # Sample points along the ray in canonical space (or observed space, 
    # depending on implementation direction. NR-NeRF uses ray bending [S2, unofficial])
    # Assuming we sample in canonical space x_c and map to observed x_o for deformation regularization
    # Or sample in observed space and map to canonical. 
    # Given "ray bending" and "canonical volume", we likely query canonical volume at deformed points.
    
    # Let's assume standard NeRF sampling in canonical space x_c
    t_vals = torch.linspace(0, 1, num_samples).to(ray_origin.device)
    x_c = ray_origin.unsqueeze(0) + ray_direction.unsqueeze(0) * t_vals.unsqueeze(-1)
    
    # 1. Get deformation
    delta_x = bending_net(x_c, t)
    
    # 2. Get rigidity
    r = rigidity_net(x_c)
    
    # 3. Apply rigidity mask
    delta_x_eff = (1 - r) * delta_x
    
    # 4. Map to observed space (if rendering requires observed coordinates)
    # Or if canonical volume is queried at x_c, and deformation is used for regularization/consistency
    # NR-NeRF implements deformation as ray bending [S2, unofficial].
    # This often implies the ray in canonical space is bent to match the observed ray.
    # For simplicity in this pseudocode, we assume we render from canonical volume
    # and the deformation is primarily for consistency and regularization.
    
    # Query canonical radiance field
    sigma, color = canonical_nerf(x_c)
    
    # Volume rendering integration (simplified)
    # ... standard alpha compositing ...
    
    return color, sigma

def train_step(batch, canonical_nerf, bending_net, rigidity_net, optimizer):
    optimizer.zero_grad()
    
    # Render images for all rays in batch
    rendered_colors, sigmas = [], []
    for ray in batch.rays:
        c, s = render_ray(ray.origin, ray.direction, batch.time, canonical_nerf, bending_net, rigidity_net)
        rendered_colors.append(c)
        sigmas.append(s)
        
    rendered_colors = torch.stack(rendered_colors)
    
    # 1. Color Loss
    loss_color = F.mse_loss(rendered_colors, batch.gt_colors)
    
    # 2. Divergence Regularization (Volume-preserving constraint) [S1, unofficial]
    # Compute divergence of the effective deformation field
    # We need to compute this over the sampled points
    # Re-compute delta_x_eff for divergence calculation
    # ... (re-run forward pass or cache) ...
    
    # Assuming we have x_c and delta_x_eff from the forward pass
    div = compute_divergence(delta_x_eff, x_c)
    loss_div = torch.mean(div ** 2)
    
    total_loss = loss_color + lambda_div * loss_div
    total_loss.backward()
    optimizer.step()
    
    return total_loss.item()
```

**Edge Cases and Pitfalls:**
*   **Occluded Areas:** Monocular input views can lead to ambiguities in occluded regions. The divergence regularization is critical to stabilize these areas [S1, unofficial].
*   **Rigid vs. Non-Rigid Boundaries:** Incorrect rigidity scores can lead to artifacts at the boundaries between rigid and non-rigid regions. The joint training must balance the rigidity constraint with the deformation accuracy [S2, unofficial].
*   **Large Deformations:** NR-NeRF is designed for small non-rigid deformations and motions [S1, unofficial]. Large topological changes may require more advanced methods like HyperNeRF [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples per ray and the depth of the MLPs. Training is iterative and requires joint optimization of three networks.
*   **Memory Complexity:** Stores the parameters of three MLPs (Canonical NeRF, Ray Bending Network, Rigidity Network). No explicit voxel grid is stored, making it memory-efficient compared to grid-based methods.
*   **Strengths:**
    *   Does not require additional supervisory cues like depth maps or scene flows [S1, unofficial].
    *   Handles rigid and non-rigid regions separately, improving stability [S2, unofficial].
    *   Allows for simple edits of the learned deformation field, such as motion exaggeration or removal of dynamic content [S1, unofficial].
*   **Limitations:**
    *   Limited to small non-rigid deformations [S1, unofficial].
    *   Controllability is severely limited compared to methods with explicit latent codes for motion [S1, unofficial].
    *   Performance on scenes with significant topological changes is not guaranteed [S2, unofficial].

## Background
Non-rigid NeRF (NR-NeRF) was introduced by Thies et al. [TTG* 21]. It builds on the original Neural Radiance Fields (NeRF) framework by incorporating a deformation field to handle dynamic scenes. The method was developed to address the limitations of static NeRF in dynamic environments without relying on external supervision [S1, unofficial]. It shares similarities with Nerfies in its use of regularization, but introduces the rigidity network as a novel component [S1, unofficial].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
