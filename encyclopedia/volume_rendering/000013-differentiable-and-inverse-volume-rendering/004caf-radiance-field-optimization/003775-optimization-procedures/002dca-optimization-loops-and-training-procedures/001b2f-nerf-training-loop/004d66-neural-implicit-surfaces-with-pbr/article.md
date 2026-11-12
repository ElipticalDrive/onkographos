# Neural Implicit Surfaces with PBR

Also called: NeRF with PBR, Ref-NeRF, VolSDF, Volume Rendering with PBR and Neural Implicit Surfaces

## Overview
This method integrates a Signed Distance Function (SDF) with a Physically-Based Rendering (PBR) material model to achieve accurate scene representation, explicitly separating specular and diffuse reflections [S1]. It differs from standard NeRF by deriving normals from the SDF gradient rather than direct optimization, and from basic SDF methods by employing a precomputed BRDF LUT for specular calculation [S1]. It builds on [NeRF Training Loop](970-nerf-training-loop.md) but replaces the generic radiance field with a geometry-aware implicit surface representation.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera poses, intrinsics, training images, and a set of 3D points sampled along rays.
*   **Outputs:** Rendered pixel colors $C(r)$, predicted volume density $\sigma$, SDF values $d$, and surface normals $\mathbf{n}$.

**Data Structures:**
*   **Spatial MLP:** An 8-layer Multi-Layer Perceptron with 256 hidden units and ReLU activations [S1].
    *   *Input:* 3D position $\mathbf{x}$.
    *   *Output:* SDF value $d(\mathbf{x})$ and volume density $\sigma(\mathbf{x})$.
*   **BRDF LUT:** A precomputed lookup table indexed by $\cos \theta$ (view angle) and roughness, containing terms $f_{prefilteredColor}$, $A$, and $B$ [S1].

**Numbered Steps:**

1.  **Ray Sampling:** Sample $N$ points along a ray $r$ within the bounding volume.
2.  **Implicit Field Query:** For each point $\mathbf{x}_i$, query the Spatial MLP to obtain the SDF value $d_i$ and volume density $\sigma_i$ [S1].
3.  **Normal Derivation:** Compute the surface normal $\mathbf{n}_i$ as the normalized gradient of the SDF:
    $$ \mathbf{n}_i = \frac{\nabla d(\mathbf{x}_i)}{\|\nabla d(\mathbf{x}_i)\|} $$
    This approach yields smoother normals compared to direct optimization constraints [S1].
4.  **Color Decomposition:** Calculate the diffuse color $C_d$ and specular color $C_s$ for each point.
    *   **Diffuse:** Typically derived from the MLP output or a separate diffuse head (not explicitly detailed in source, but implied by decomposition) [S1].
    *   **Specular:** Calculate using the PBR model:
        $$ C_s = f_{prefilteredColor} * (F_0 * A + B) $$
        where $F_0$ is the reflectance at a perpendicular angle, and $f_{prefilteredColor}$, $A$, and $B$ are retrieved from the precomputed BRDF LUT based on view angle and roughness [S1].
    *   **Total Radiance:** $c_i = C_d + C_s$ [S1].
5.  **Volume Rendering:** Compute the final pixel color $C(r)$ using the discrete volume rendering equation:
    $$ C(r) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
    where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the cumulative transmittance, $\sigma_i$ is the volume density, and $\delta_i$ is the segment length [S1].
6.  **Loss Calculation:** Compute the total loss $\mathcal{L}$:
    $$ \mathcal{L} = \mathcal{L}_{RGB} + \lambda \mathcal{L}_{SDF} $$
    *   **Photometric Loss:** $\mathcal{L}_{RGB} = \sum_{r \in R} \hat{C}(r) - C(r)$, where $\hat{C}(r)$ is the ground truth color [S1].
    *   **Eikonal Loss:** $\mathcal{L}_{SDF} = \sum_{p \in \mathbb{R}^3} (\nabla d(p) - 1)^2$, encouraging the SDF gradient magnitude to be 1 [S1].
7.  **Optimization:** Update network weights using gradient descent [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class NeuralImplicitPBR(nn.Module):
    def __init__(self):
        super().__init__()
        # Spatial MLP: 8 layers, 256 hidden units, ReLU
        layers = []
        for i in range(8):
            layers.append(nn.Linear(256 if i > 0 else 3, 256))
            layers.append(nn.ReLU())
        self.mlp = nn.Sequential(*layers)
        # Output: SDF (1) and Density (1)
        self.output_layer = nn.Linear(256, 2)

    def forward(self, x):
        """
        x: (N, 3) tensor of 3D points
        Returns: sdf (N,), sigma (N,)
        """
        h = self.mlp(x)
        out = self.output_layer(h)
        sdf = out[:, 0:1]
        sigma = out[:, 1:2]
        return sdf.squeeze(-1), sigma.squeeze(-1)

    def get_normal(self, x, sdf):
        """
        Compute normal from SDF gradient using finite differences or autograd.
        Using autograd for precision.
        """
        x.requires_grad_(True)
        sdf_val = self.forward(x)[0]
        grad = torch.autograd.grad(
            sdf_val.sum(), x, create_graph=True, retain_graph=True
        )[0]
        normal = F.normalize(grad, dim=-1)
        return normal

    def compute_specular(self, normal, view_dir, roughness, F0):
        """
        Compute specular color using precomputed BRDF LUT terms.
        Note: In a real implementation, LUT lookup replaces the formula below.
        Here we simulate the structure described: Cs = f_prefilteredColor * (F0 * A + B)
        """
        # Cosine of the angle between normal and view direction
        cos_theta = torch.clamp(torch.sum(normal * view_dir, dim=-1), min=0.0, max=1.0)
        
        # Placeholder for LUT lookup logic. 
        # In practice, f_prefilteredColor, A, B are tensors indexed by cos_theta and roughness.
        # For this self-contained example, we assume these are pre-computed or passed in.
        # The source defines the structure: Cs = f_prefilteredColor * (F0 * A + B)
        # We assume f_prefilteredColor, A, B are available tensors of shape (N,)
        # or looked up from a static LUT.
        
        # Since the source does not provide the exact LUT interpolation code,
        # we represent the formula as given.
        # f_prefiltered_color, A, B would be retrieved from LUT(cos_theta, roughness)
        pass 

    def volume_render(self, sigma, delta, color):
        """
        Discrete volume rendering.
        sigma: (N,) density
        delta: (N,) segment length
        color: (N, 3) radiance
        Returns: (3,) final color
        """
        transmittance = torch.exp(-torch.cumsum(sigma * delta, dim=0)[:-1])
        # Add initial transmittance of 1.0
        transmittance = torch.cat([torch.ones_like(transmittance[:1]), transmittance], dim=0)
        
        alpha = 1.0 - torch.exp(-sigma * delta)
        weights = alpha * transmittance
        final_color = torch.sum(weights.unsqueeze(-1) * color, dim=0)
        return final_color

    def compute_loss(self, pred_color, gt_color, sdf, x):
        """
        Compute total loss: L_RGB + lambda * L_SDF
        """
        # Photometric loss
        l_rgb = torch.mean((pred_color - gt_color) ** 2)
        
        # Eikonal loss
        # Recompute gradient for Eikonal loss on sampled points
        x.requires_grad_(True)
        sdf_val = self.forward(x)[0]
        grad = torch.autograd.grad(
            sdf_val.sum(), x, create_graph=True, retain_graph=True
        )[0]
        grad_norm = torch.norm(grad, dim=-1)
        l_sdf = torch.mean((grad_norm - 1.0) ** 2)
        
        lambda_sdf = 1.0 # Typical value, not specified in source
        total_loss = l_rgb + lambda_sdf * l_sdf
        
        return total_loss
```

**Parameters:**
*   **MLP Architecture:** 8 layers, 256 hidden units, ReLU activations [S1].
*   **Hardware:** Ubuntu 20.04, single NVIDIA RTX3060 GPU (12GB memory) [S1].
*   **Lambda ($\lambda$):** Weight for Eikonal loss. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sharp Irregularities:** The SDF representation favors smooth geometric shapes, which is a disadvantage for scenes with sharp irregularities (e.g., 'ficus') [S1].
*   **Refraction:** The model does not accurately model refraction and transmission phenomena, likely because the SDF geometric representation tends to utilize a single surface layer [S1].

## Operating characteristics
*   **Performance:** On the 'sedan' scene, the method achieves a PSNR of 25.92 and SSIM of 0.722, outperforming Ref-NeRF (25.65/0.720), VolSDF (21.70/0.583), and NeRF (20.08/0.564) [S1].
*   **Strengths:** Produces smoother geometric shapes and accurately separates specular and diffuse reflections, improving view-dependent reflection components [S1].
*   **Limitations:** Struggles with scenes containing sharp irregularities (e.g., 'ficus') due to the smoothness bias of SDFs [S1]. Cannot model refraction/transmission [S1].
*   **Complexity:** Not given in the sources.

## Background
This method was introduced in a paper titled "Volume rendering based on PBR and neural implicit surfaces" [S1]. It builds upon the VolSDF framework and incorporates PBR theory to modulate specular reflections and diffuse components [S1]. The work addresses the challenge of achieving accurate mirror reflections and geometric representation in neural radiance fields.

## Sources
[S1] Volume rendering based on PBR and neural implicit surfaces, https://doi.org/10.1145/3650400.3650464, fetched 2026-10-27
