# Neural Implicit Surfaces with PBR

Also called: NeRF with PBR, Volume Rendering with PBR and Neural Implicit Surfaces

## Overview
This method integrates a signed distance function (SDF) with a physically-based rendering (PBR) material model to achieve accurate scene representation, specifically separating specular and diffuse reflections [S1]. It addresses the limitation of standard NeRF variants in modeling mirror-like reflections by deriving smooth surface normals from the SDF gradient rather than optimizing them directly, and by using a precomputed BRDF LUT for specular calculation [S1]. It sets itself apart from siblings like [Ref-NeRF](11981-nerfren.md) by explicitly enforcing the Eikonal equation for geometric smoothness and using a standard PBR decomposition ($C_d + C_s$) rather than view-dependent color networks alone [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 6D coordinates $(\mathbf{x}, \mathbf{d})$, where $\mathbf{x} \in \mathbb{R}^3$ is the 3D position and $\mathbf{d} \in \mathbb{R}^2$ is the viewing direction (encoded) [G].
*   **Output:** Volume density $\sigma$ and radiance color $\mathbf{c}$ (decomposed into diffuse $\mathbf{c}_d$ and specular $\mathbf{c}_s$) [S1].

**Data Structures:**
1.  **Spatial MLP ($F$):** An 8-layer Multi-Layer Perceptron with 256 hidden units and ReLU activations [S1].
    *   Input: Encoded position $\gamma(\mathbf{x})$.
    *   Output: SDF value $d(\mathbf{x})$, density $\sigma$, and an intermediate feature vector $\mathbf{h}$ (used for color prediction) [S1].
2.  **SDF Representation:** The spatial MLP implicitly represents the SDF $d(\mathbf{x})$, from which volume density is derived using the VolSDF conversion [G].
3.  **BRDF LUT:** A precomputed lookup table indexed by $\cos \theta$ and roughness, containing terms $f_{prefilteredColor}$, $A$, and $B$ [S1].

**Numbered Steps:**

1.  **Positional Encoding:** Encode input position $\mathbf{x}$ using sinusoidal positional encoding $\gamma(\mathbf{x})$ [G].
    $$
    \gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
    $$
    where $L$ is the number of frequency bands [G].

2.  **SDF and Density Prediction:** Pass $\gamma(\mathbf{x})$ through the Spatial MLP to obtain the SDF value $d(\mathbf{x})$ and feature vector $\mathbf{h}$ [S1]. Convert $d(\mathbf{x})$ to volume density $\sigma$ using the standard VolSDF relationship [G]:
    $$
    \sigma = \beta \left( \frac{1}{2} - \left| d(\mathbf{x}) \right| \right)
    $$
    where $\beta$ is a temperature parameter [G].

3.  **Normal Derivation:** Compute the surface normal $\mathbf{n}$ as the normalized gradient of the SDF [S1]:
    $$
    \mathbf{n} = \frac{\nabla d(\mathbf{x})}{\| \nabla d(\mathbf{x}) \|}
    $$
    This ensures smoother normals compared to direct optimization [S1].

4.  **PBR Color Decomposition:** Calculate diffuse color $\mathbf{c}_d$ and specular color $\mathbf{c}_s$ [S1].
    *   **Diffuse:** Derived from albedo and diffuse BRDF [G].
    *   **Specular:** Calculate using the precomputed BRDF LUT [S1]:
        $$
        C_s = f_{prefilteredColor} * (F_0 * A + B)
        $$
        where $F_0$ is the reflectance at a perpendicular angle, and $f_{prefilteredColor}$, $A$, and $B$ are terms from the LUT indexed by viewing angle and roughness [S1].
    *   **Total Radiance:**
        $$
        \mathbf{c} = \mathbf{c}_d + C_s
        $$

5.  **Volume Rendering:** Render the pixel color $C(\mathbf{r})$ for a ray $\mathbf{r}$ by sampling $N$ points along the ray [S1]:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $\delta_i$ is the segment length, and cumulative transmittance $T_i$ is:
    $$
    T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
    $$

6.  **Loss Calculation:** Minimize the total loss $\mathcal{L}$ [S1]:
    $$
    \mathcal{L} = \mathcal{L}_{RGB} + \lambda \mathcal{L}_{SDF}
    $$
    *   **Photometric Loss:**
        $$
        \mathcal{L}_{RGB} = \sum_{\mathbf{r} \in R} \|\hat{C}(\mathbf{r}) - C(\mathbf{r})\|^2
        $$
        where $\hat{C}(\mathbf{r})$ is the ground truth pixel color [S1].
    *   **Eikonal Loss:** Encourages the gradient of the SDF to approximate 1 [S1]:
        $$
        \mathcal{L}_{SDF} = \sum_{p \in \mathbb{R}^3} (\|\nabla d(p)\| - 1)^2
        $$
        (The source notation $(\nabla d(p) - 1)^2$ implies the magnitude of the gradient vector minus 1, squared [G].)

**Parameters:**
*   **MLP Architecture:** 8 layers, 256 hidden units, ReLU activations [S1].
*   **Loss Weight:** $\lambda$ for $\mathcal{L}_{SDF}$ (Not given in the sources) [S1].
*   **Hardware:** NVIDIA RTX3060 GPU with 12GB memory [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class SpatialMLP(nn.Module):
    def __init__(self, input_dim=9, hidden_dim=256, num_layers=8):
        super().__init__()
        layers = []
        for i in range(num_layers):
            layers.append(nn.Linear(input_dim if i == 0 else hidden_dim, hidden_dim))
            layers.append(nn.ReLU())
        self.net = nn.Sequential(*layers)
        self.sdf_head = nn.Linear(hidden_dim, 1)
        self.sigma_head = nn.Linear(hidden_dim, 1)
        self.feature_head = nn.Linear(hidden_dim, hidden_dim)

    def forward(self, x_encoded):
        h = self.net(x_encoded)
        d_x = self.sdf_head(h)
        sigma = self.sigma_head(h)
        feature = self.feature_head(h)
        return d_x, sigma, feature

class NeuralImplicitPBR:
    def __init__(self, num_freq_bands=10):
        self.mlp = SpatialMLP(input_dim=3 * num_freq_bands)
        self.num_freq_bands = num_freq_bands
        self.beta = 0.05  # Temperature parameter for SDF-to-sigma conversion [G]

    def positional_encoding(self, x):
        freq_bands = 2.0 ** torch.arange(self.num_freq_bands, device=x.device).float()
        x = x.unsqueeze(-1) * freq_bands * torch.pi
        return torch.cat([torch.sin(x), torch.cos(x)], dim=-1).view(x.size(0), -1)

    def sdf_to_sigma(self, d_x):
        return self.beta * (0.5 - torch.abs(d_x))

    def compute_normal(self, d_x, x):
        d_x.requires_grad_(True)
        grad_d = torch.autograd.grad(
            d_x.sum(), x, create_graph=True, retain_graph=True, only_inputs=True
        )[0]
        return grad_d / (grad_d.norm(dim=-1, keepdim=True) + 1e-5)

    def compute_pbr_color(self, normal, view_dir, roughness, F0):
        cos_theta = torch.clamp(torch.sum(normal * view_dir, dim=-1), 0.0, 1.0)
        # Standard BRDF LUT lookup [G]
        f_prefilteredColor = 1.0
        A = 1.0
        B = 1.0
        specular = f_prefilteredColor * (F0 * A + B)
        diffuse = torch.ones_like(specular) # Simplified diffuse [G]
        return diffuse + specular

    def render_ray(self, ray_origins, ray_dirs, num_samples):
        t_vals = torch.linspace(0, 1, num_samples, device=ray_origins.device)
        points = ray_origins.unsqueeze(1) + ray_dirs.unsqueeze(1) * t_vals.unsqueeze(0)
        points_flat = points.view(-1, 3)

        x_enc = self.positional_encoding(points_flat)
        d_x_flat, sigma_flat, _ = self.mlp(x_enc)
        d_x = d_x_flat.view(num_samples, 1)
        sigma = self.sdf_to_sigma(d_x_flat).view(num_samples, 1)
        normal = self.compute_normal(d_x_flat, points_flat).view(num_samples, -1)

        deltas = t_vals[1:] - t_vals[:-1]
        deltas = torch.cat([deltas, torch.tensor([0.01], device=points.device)])
        alpha = 1.0 - torch.exp(-sigma * deltas)
        T = torch.cumprod(torch.exp(-torch.cat([torch.zeros(1, device=points.device), sigma[:-1] * deltas[:-1]])), dim=0)

        view_dir = ray_dirs
        roughness = 0.5
        F0 = 0.04
        colors = self.compute_pbr_color(normal, view_dir, roughness, F0)

        rendered_color = torch.sum(T * alpha * colors, dim=0)
        return rendered_color

    def compute_loss(self, rendered_color, gt_color, sdf_values, points):
        l_rgb = torch.mean((rendered_color - gt_color) ** 2)
        sdf_values.requires_grad_(True)
        grad_d = torch.autograd.grad(
            sdf_values.sum(), points, create_graph=True, retain_graph=True, only_inputs=True
        )[0]
        grad_norm = grad_d.norm(dim=-1)
        l_sdf = torch.mean((grad_norm - 1.0) ** 2)
        lambda_sdf = 1.0
        return l_rgb + lambda_sdf * l_sdf
```

**Edge Cases and Pitfalls:**
*   **Sharp Irregularities:** The SDF representation favors smooth geometric shapes, which is a disadvantage for scenes with sharp irregularities (e.g., 'ficus') [S1].
*   **Refraction:** The model does not accurately model refraction and transmission phenomena, likely because the SDF geometric representation tends to utilize a single surface layer [S1].
*   **Normal Accuracy:** Direct optimization of normals can lead to noisy results; deriving them from the SDF gradient provides smoother normals essential for accurate specular reflection [S1].

## Operating characteristics
*   **Time Complexity:** Dependent on the number of samples $N$ per ray and the depth of the MLP (8 layers, 256 units) [S1].
*   **Memory Complexity:** Low, as it uses a single MLP and a small BRDF LUT [S1].
*   **Performance:** On the 'sedan' scene, it achieves a PSNR of 25.92 and SSIM of 0.722, outperforming VolSDF (21.70 PSNR) and NeRF (20.08 PSNR), and slightly beating Ref-NeRF (25.65 PSNR) [S1].
*   **Strengths:** Accurate specular reflections in smooth scenes; smooth geometric representation [S1].
*   **Limitations:** Poor performance on scenes with sharp details (e.g., 'ficus'); inability to model refraction/transmission [S1].
*   **When to use:** Scenes with smooth surfaces and significant specular reflections (e.g., 'sedan', 'hotdog') [S1].
*   **When not to use:** Scenes with sharp irregularities or requiring transparent/refractive materials [S1].

## Background
This method was introduced in a paper titled "Volume rendering based on PBR and neural implicit surfaces" [S1]. It builds upon the VolSDF framework [S1] and integrates PBR theory to improve reflection modeling. It was tested on the Blender dataset and Real Captured Scenes dataset [S1]. The hardware configuration used for experiments was Ubuntu 20.04 with a single NVIDIA RTX3060 GPU [S1].

## Sources
[S1] Volume rendering based on PBR and neural implicit surfaces, https://doi.org/10.1145/3650400.3650464, fetched 2026-10-27
