# GSurf SDF Learning from Gaussians
Also called: Learning SDF from Gaussians

## Overview
GSurf is a method for fitting a Signed Distance Field (SDF) directly to the centroids of 3D Gaussian primitives to provide geometric regularization and rich geometric cues for appearance modeling [S1]. It addresses holes and surface irregularities common in TSDF fusion by leveraging explicit 3D supervision from Gaussians, unlike NeuS-based methods which rely purely on 2D image supervision [S1]. The method ensures that the majority of Gaussian disks have high opacity to align closely with the surface, distinguishing it from vanilla 2DGS/3DGS which contain many semi-transparent primitives [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussian primitives $\{G_k\}$, each defined by a centroid position $\mathbf{p}_k$, an orientation $\mathbf{n}_k$ (defined by the shortest axis of the disk), and an opacity $\alpha_k$. Multi-view images are used for photometric loss.
*   **Outputs:** An SDF field $f_{sdf}(\mathbf{x})$ parameterized by an MLP, and updated Gaussian parameters (positions, orientations, opacities, colors).

**Data Structures:**
*   **SDF MLP:** An 8-layer Multi-Layer Perceptron with 256 hidden units and ReLU activation [S1].
*   **Appearance MLP:** A 4-layer MLP with 256 hidden units and ReLU activation, used to predict color $c_k$ based on position $\mathbf{p}_k$, depth $d$, SDF gradient $\nabla f_{sdf}$, and geometry features $F_{geo}$ [S1].
*   **Gaussians:** Standard 3D Gaussian splatting representation, but constrained to high opacity [S1].

**Parameters:**
*   Loss weights: $\lambda_1 = 0.1$, $\lambda_2 = 0.01$, $\lambda_3 = 0.01$, $\lambda_4 = 0.05$, $\lambda_5 = 0.05$ [S1].
*   Off-surface constraint coefficient: $\alpha = 100$ [S1].
*   Delayed activation: SDF learning begins after 500 iterations [S1].

**Steps:**

1.  **Initialization:**
    *   Initialize the SDF MLP as a sphere [S1].
    *   Initialize Gaussians randomly (for OmniObjects3D) or from sparse, noisy Structure-from-Motion (SfM) point clouds (for DTU) [S1].
    *   Set a flag `sdf_active = False`.

2.  **Warm-up Phase (Iterations 1–500):**
    *   If `sdf_active` is False:
        *   Compute multi-view photometric loss using the appearance MLP and Gaussian splatting [S1].
        *   Apply entropy-based opacity regularization to prune severe outliers [S1].
        *   Update Gaussian parameters and Appearance MLP using Adam optimizer [S1].
        *   Do not compute SDF losses [S1].
    *   After iteration 500, set `sdf_active = True` [S1].

3.  **SDF Learning Phase (Iteration > 500):**
    *   For each Gaussian centroid $\mathbf{p}_k$:
        *   Compute position loss $L_{pos}$:
            $$ L_{pos} = \sum_k |f_{sdf}(\mathbf{p}_k)| $$
            This encourages the zero-level set to align with the Gaussians [S1].
        *   Compute orientation loss $L_{ori}$:
            $$ L_{ori} = 1 - \mathbf{n}_k \cdot \frac{\nabla f_{sdf}(\mathbf{p}_k)}{\|\nabla f_{sdf}(\mathbf{p}_k)\|} $$
            This enforces alignment between the Gaussian orientation $\mathbf{n}_k$ and the SDF gradient [S1].
    *   Sample a set of points $\chi_{eik}$ in space:
        *   Compute Eikonal loss $L_{eik}$:
            $$ L_{eik} = \sum_{\mathbf{x} \in \chi_{eik}} (\|\nabla f_{sdf}(\mathbf{x})\|^2 - 1)^2 $$
            This enforces unit gradient norm and promotes SDF smoothness [S1].
    *   Sample a set of query points $\{\mathbf{q}_k\}$ uniformly in space:
        *   Compute off-surface loss $L_{off}$:
            $$ L_{off} = \sum_k \exp(-\alpha |f_{sdf}(\mathbf{q}_k)|) $$
            This keeps function values for off-surface points relatively large, away from zero [S1].
    *   Render normal maps from the SDF ($N_{sdf}$) and from the 2DGS representation ($N_{gs}$):
        *   Compute normal map loss $L_{nor}$:
            $$ L_{nor} = 1 - N_{gs} \cdot N_{sdf} $$
            This minimizes the discrepancy between the normal maps [S1].
    *   Compute total SDF loss:
        $$ L_{sdf} = \lambda_1 L_{pos} + \lambda_2 L_{eik} + \lambda_3 L_{off} + \lambda_4 L_{ori} + \lambda_5 L_{nor} $$
        [S1].
    *   Compute total loss:
        $$ L_{total} = L_{gs} + L_{sdf} + L_{ent} $$
        where $L_{gs}$ is the Gaussian splatting photometric loss and $L_{ent}$ is the entropy-based opacity regularization [S1].
    *   Update SDF MLP, Appearance MLP, and Gaussian parameters using Adam optimizer [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

class SDFMLP(nn.Module):
    def __init__(self):
        super().__init__()
        layers = []
        for i in range(8):
            layers.append(nn.Linear(256, 256))
            layers.append(nn.ReLU())
        self.net = nn.Sequential(*layers)
        # Note: Source specifies 8 layers, 256 hidden units. 
        # Standard MLP structure implies input dim 3, output dim 1.
        # The first layer takes 3D coordinates.
        self.net = nn.Sequential(
            nn.Linear(3, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 1)
        )

    def forward(self, x):
        return self.net(x).squeeze(-1)

def compute_gradient(f, x):
    """Compute gradient of f with respect to x."""
    x.requires_grad_(True)
    y = f(x)
    grad = torch.autograd.grad(y, x, grad_outputs=torch.ones_like(y), create_graph=True)[0]
    return grad

def train_gsurf(gaussians, images, num_iterations=10000):
    sdf_mlp = SDFMLP()
    appearance_mlp = AppearanceMLP() # 4 layers, 256 hidden, ReLU
    optimizer_sdf = optim.Adam(sdf_mlp.parameters(), lr=1e-4)
    optimizer_appearance = optim.Adam(appearance_mlp.parameters(), lr=1e-4)
    optimizer_gaussians = optim.Adam(gaussians.parameters(), lr=1e-3)

    # Hyperparameters
    lambda_pos = 0.1
    lambda_eik = 0.01
    lambda_off = 0.01
    lambda_ori = 0.05
    lambda_nor = 0.05
    alpha_off = 100
    
    sdf_active = False

    for i in range(num_iterations):
        # Warm-up phase
        if i < 500:
            # Only photometric and opacity regularization
            loss_gs = compute_photometric_loss(gaussians, images)
            loss_ent = compute_entropy_regularization(gaussians)
            loss_total = loss_gs + loss_ent
            
            optimizer_gaussians.zero_grad()
            optimizer_appearance.zero_grad()
            loss_total.backward()
            optimizer_gaussians.step()
            optimizer_appearance.step()
            continue
        
        sdf_active = True
        
        # SDF Learning
        optimizer_sdf.zero_grad()
        optimizer_appearance.zero_grad()
        optimizer_gaussians.zero_grad()
        
        # 1. Position Loss
        p_k = gaussians.centroids # Shape: [N, 3]
        sdf_vals_p = sdf_mlp(p_k)
        loss_pos = torch.mean(torch.abs(sdf_vals_p))
        
        # 2. Orientation Loss
        grad_p = compute_gradient(sdf_mlp, p_k) # Shape: [N, 3]
        grad_norm_p = torch.norm(grad_p, dim=1, keepdim=True)
        grad_unit_p = grad_p / (grad_norm_p + 1e-8)
        n_k = gaussians.normals # Shape: [N, 3], shortest axis of disk
        loss_ori = torch.mean(1 - torch.sum(n_k * grad_unit_p, dim=1))
        
        # 3. Eikonal Loss
        # Sample points chi_eik uniformly in space
        x_eik = sample_uniform_points(num_samples=10000)
        grad_x = compute_gradient(sdf_mlp, x_eik)
        grad_norm_x = torch.norm(grad_x, dim=1)
        loss_eik = torch.mean((grad_norm_x**2 - 1)**2)
        
        # 4. Off-surface Loss
        q_k = sample_uniform_points(num_samples=10000)
        sdf_vals_q = sdf_mlp(q_k)
        loss_off = torch.mean(torch.exp(-alpha_off * torch.abs(sdf_vals_q)))
        
        # 5. Normal Map Loss
        # Render normal maps from SDF and 2DGS
        N_sdf = render_normal_map_from_sdf(sdf_mlp, camera_params)
        N_gs = render_normal_map_from_gaussians(gaussians, camera_params)
        loss_nor = torch.mean(1 - torch.sum(N_gs * N_sdf, dim=1))
        
        # Total SDF Loss
        loss_sdf = (lambda_pos * loss_pos + 
                    lambda_eik * loss_eik + 
                    lambda_off * loss_off + 
                    lambda_ori * loss_ori + 
                    lambda_nor * loss_nor)
        
        # Total Loss
        loss_gs = compute_photometric_loss(gaussians, images)
        loss_ent = compute_entropy_regularization(gaussians)
        loss_total = loss_gs + loss_sdf + loss_ent
        
        loss_total.backward()
        optimizer_sdf.step()
        optimizer_appearance.step()
        optimizer_gaussians.step()
```

**Edge Cases and Pitfalls:**
*   **Cold-start divergence:** At initialization, Gaussian primitives from raw point clouds are chaotic and noisy. Supervising the implicit field with these unaligned centroids pulls the zero-level set away from the true surface, causing geometric distortion and optimization collapse [S1].
*   **Mitigation:** A 500-iteration delayed SDF activation strategy is used. During this warm-up phase, only the multi-view photometric loss and entropy-based opacity regularization operate to prune severe outliers [S1].
*   **Opacity Constraint:** Unlike vanilla 2DGS/3DGS, GSurf requires the majority of Gaussian disks to have high opacity to align closely with the surface [S1].

## Operating characteristics
*   **Time Complexity:** Training time is comparable to other Gaussian-SDF methods. Prior methods typically require 2-16 hours (e.g., 2 hours for GSDF, up to 16 hours for NeuSG) [S1]. GSurf is trained on a single NVIDIA A800 GPU [S1].
*   **Memory Complexity:** Requires memory for the SDF MLP (8 layers, 256 hidden units), Appearance MLP (4 layers, 256 hidden units), and Gaussian primitives [S1].
*   **Strengths:** Provides explicit 3D supervision via Gaussian centroids, similar to surface reconstruction from point clouds, unlike NeuS-based methods which are purely trained from 2D images [S1]. The learned SDF provides rich geometric cues that enhance Gaussian appearance modeling, unlike TSDF-based methods that rely solely on opacity and spherical harmonics [S1].
*   **Limitations:** Requires careful initialization and warm-up to avoid cold-start divergence [S1]. Relies on high-opacity Gaussians, which may be challenging for transparent or thin structures [S1].
*   **When to use:** When high-quality 3D reconstruction with geometric regularization is needed, and explicit 3D supervision from Gaussians is available. Particularly effective for opaque objects [S1].
*   **When not to use:** For scenes with significant transparency or where Gaussian primitives cannot be constrained to high opacity [S1].

## Background
GSurf was proposed in the paper "GSurf: Learning signed distance fields from splatting opaque Gaussians for high-quality 3D reconstruction" [S1]. It builds on recent advances in shape representation using SDFs and their application in neural rendering (e.g., NeuS, VolSDF) [S1]. The method modifies 2DGS by using an MLP to predict color instead of spherical harmonics and by requiring high opacity for Gaussian disks [S1]. It was evaluated on the DTU dataset and OmniObject3D datasets [S1].

## Sources
[S1] GSurf: Learning signed distance fields from splatting opaque Gaussians for high-quality 3D reconstruction, https://doi.org/10.1016/j.cad.2026.104106, fetched 2026-10-26
