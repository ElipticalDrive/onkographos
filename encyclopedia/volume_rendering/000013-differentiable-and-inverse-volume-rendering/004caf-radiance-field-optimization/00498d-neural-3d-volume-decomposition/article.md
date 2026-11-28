# Neural 3D Volume Decomposition
## Overview
Neural 3D Volume Decomposition (Neural-PIL) is a method for jointly estimating 3D shape, spatially-varying BRDF (SVBRDF), and per-image illumination from a collection of images of an object captured under varying viewpoints and lighting conditions [S1, unofficial]. It extends the Neural Radiance Field (NeRF) framework by replacing the implicit view-dependent color output with an explicit physical material representation (Cook-Torrance BRDF parameters) and a latent illumination vector [S1, unofficial]. Unlike standard NeRF which optimizes for novel view synthesis directly, this method solves the inverse problem of reflectance decomposition, enabling high-quality relighting and material-aware view synthesis [S1, unofficial].

## How it works
The method builds on [Radiance-field optimization](44-radiance-field-optimization.md) by introducing a physically-based rendering formulation within the neural volume [S1, unofficial]. It uses a two-stage sampling strategy similar to NeRF but optimizes for material and lighting parameters rather than just color [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   A set of $q$ images $C_j \in \mathbb{R}^{s \times 3}$ for $j \in \{1, \dots, q\}$, where $s$ is the number of pixels per image [S1, unofficial].
*   Per-pixel binary masks $M_j \in \{0, 1\}^{s \times 1}$ indicating which pixels belong to the object [S1, unofficial].

**Outputs:**
*   A neural 3D volume $V$ that, for any 3D point $x \in \mathbb{R}^3$, estimates:
    *   Cook-Torrance BRDF parameters $b \in \mathbb{R}^7$: diffuse color $b_d \in \mathbb{R}^3$, specular color $b_s \in \mathbb{R}^3$, and roughness $b_r \in \mathbb{R}$ [S1, unofficial].
    *   Unit-length surface normal $n \in \mathbb{R}^3$ [S1, unofficial].
    *   Optical density $\sigma \in \mathbb{R}$ [S1, unofficial].
*   Per-image latent illumination vectors $z_l \in \mathbb{R}^{128}$ for each of the $q$ images [S1, unofficial].

### Data Structures
The neural volume $V$ is implemented as two Multi-Layer Perceptrons (MLPs) [S1, unofficial].
*   **Coarse MLP:** Learns a coarse representation and samples the 3D volume in a fixed sampling pattern [S1, unofficial].
*   **Fine MLP:** Uses the coarse density knowledge to sample the volume in high-density areas at a finer resolution [S1, unofficial].
*   **Input to MLPs:** 3D coordinate $x \in \mathbb{R}^3$ and view direction $d \in \mathbb{R}^3$ [S1, unofficial].
*   **Output of MLPs:** Predicts material and density parameters in a fixed tensor layout: indices 0-2 for $b_d$, 3-5 for $b_s$, 6 for $b_r$, 7-9 for $n$, and 10 for $\sigma$ [S1, unofficial].
*   **Illumination Storage:** A learnable tensor of shape $q \times 128$ storing $z_l$ for each image [S1, unofficial].

### Rendering Formulation
The method replaces NeRF's direct color prediction with a rendering equation that integrates BRDF and illumination [S1, unofficial].

1.  **Ray Sampling:** For a camera ray $r(t) = o + td$, sample points $t_i$ along the ray between near bound $t_n$ and far bound $t_f$ [S1, unofficial].
2.  **Neural Query:** For each sample point $t_i$, query the neural volume $V$ to get $b(t_i)$, $n(t_i)$, and $\sigma(t_i)$ [S1, unofficial].
3.  **Illumination Retrieval:** Retrieve the latent illumination vector $z_l$ corresponding to the image index $j$ of the camera ray [S1, unofficial].
4.  **BRDF Evaluation:** Compute the Cook-Torrance BRDF $f_r$ using $b(t_i)$, the surface normal $n(t_i)$, the view direction $d$, and the light direction derived from $z_l$ [S1, unofficial].
5.  **Radiance Integration:** Compute the outgoing radiance $L_o$ at each sample point by integrating the incident illumination against the BRDF [S1, unofficial].
6.  **Volume Rendering:** Composite the final color $\hat{c}(r)$ using numerical quadrature [S1, unofficial]:

$$
\hat{c}(r) = \int_{t_n}^{t_f} T(t) \sigma(t) c(t) dt
$$

with $T(t) = \exp\left(-\int_{t_n}^{t} \sigma(t') dt'\right)$ [S1, unofficial]. Here, $c(t)$ represents the outgoing radiance computed from the BRDF and illumination at sample $t$.

### Optimization
The network is optimized by minimizing the difference between the rendered images and the input images $C_j$ [S1, unofficial]. The loss function typically includes a photometric term comparing $\hat{c}(r)$ to the pixel color in $C_j$ for pixels where $M_j=1$ [S1, unofficial]. The method may also use pre-trained networks (neural-PIL, light-SMAE, BRDF-SMAE) to regularize or initialize the optimization of BRDF and light parameters [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class NeuralVolumeDecomposition:
    def __init__(self, num_images, coarse_samples=64, fine_samples=128, near=0.1, far=10.0):
        self.num_images = num_images
        self.coarse_samples = coarse_samples
        self.fine_samples = fine_samples
        self.near = near
        self.far = far
        # Coarse MLP: maps (x, d) to material/density parameters
        self.coarse_mlp = nn.Sequential(
            nn.Linear(6, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 11)
        )
        # Fine MLP: refines sampling in high-density areas
        self.fine_mlp = nn.Sequential(
            nn.Linear(6, 256), nn.ReLU(),
            nn.Linear(256, 256), nn.ReLU(),
            nn.Linear(256, 11)
        )
        # Learnable per-image illumination latent vectors z_l in R^128
        self.illumination_latents = nn.Parameter(torch.randn(num_images, 128))

    def sample_rays(self, origins, directions, num_samples):
        t_vals = torch.linspace(self.near, self.far, num_samples, device=origins.device)
        origins_exp = origins.unsqueeze(1)
        dirs_exp = directions.unsqueeze(1)
        t_vals_exp = t_vals.unsqueeze(0).unsqueeze(2)
        points = origins_exp + dirs_exp * t_vals_exp
        deltas = torch.diff(t_vals, dim=0)
        delta_inf = torch.tensor([self.far - t_vals[-1]], device=origins.device)
        deltas = torch.cat([deltas, delta_inf], dim=0)
        deltas = deltas.unsqueeze(0).unsqueeze(0).expand(origins.shape[0], num_samples, 1)
        return points, deltas

    def query_volume(self, points, directions, mlp):
        inputs = torch.cat([points, directions], dim=-1)
        outputs = mlp(inputs)
        b_d = outputs[:, :, 0:3]
        b_s = outputs[:, :, 3:6]
        b_r = outputs[:, :, 6:7]
        n = outputs[:, :, 7:10]
        sigma = outputs[:, :, 10:11]
        n = n / (torch.norm(n, dim=-1, keepdim=True) + 1e-6)
        return b_d, b_s, b_r, n, sigma

    def render(self, origins, directions, image_idx):
        # Stage 1: Coarse fixed sampling
        coarse_points, coarse_deltas = self.sample_rays(origins, directions, self.coarse_samples)
        _, _, _, _, coarse_sigma = self.query_volume(coarse_points, directions, self.coarse_mlp)
        coarse_sigma = torch.clamp(coarse_sigma, min=0.0)

        # Stage 2: Fine sampling (density-guided refinement [G])
        fine_points, fine_deltas = self.sample_rays(origins, directions, self.fine_samples)
        fine_b_d, fine_b_s, fine_b_r, fine_n, fine_sigma = self.query_volume(fine_points, directions, self.fine_mlp)
        fine_sigma = torch.clamp(fine_sigma, min=0.0)

        # Use fine samples for final composition
        points = fine_points
        deltas = fine_deltas
        b_d, b_s, b_r, n, sigma = fine_b_d, fine_b_s, fine_b_r, fine_n, fine_sigma

        # Compute outgoing radiance c(t) from BRDF and illumination
        # The exact mapping from z_l to light directions/intensity uses pre-trained networks (neural-PIL, light-SMAE) [G]
        z_l = self.illumination_latents[image_idx]
        c = torch.zeros_like(b_d)

        alpha = 1.0 - torch.exp(-sigma.squeeze(-1) * deltas.squeeze(-1))
        T = torch.exp(-torch.cumsum(sigma.squeeze(-1) * deltas.squeeze(-1), dim=1))
        T_shifted = torch.cat([torch.ones(origins.shape[0], 1, device=origins.device), T[:, :-1]], dim=1)
        weights = T_shifted * alpha.unsqueeze(-1)
        rendered_color = torch.sum(weights * c, dim=1)
        return rendered_color
```

### Edge Cases and Pitfalls
*   **Precision:** The optimization of BRDF parameters (especially roughness $b_r$) can be sensitive to initialization. Pre-training with neural-PIL helps stabilize this [S1, unofficial].
*   **Boundaries:** The near and far bounds $t_n$ and $t_f$ must be chosen to encompass the object. If the object is outside these bounds, it will not be rendered [S1, unofficial].

## Subtopics

### Hierarchical Grid Optimization
Accelerates neural volume queries using a multi-resolution grid structure, reducing memory and computation compared to dense MLPs [1414-hierarchical-grid-optimization.md]. Details on integration with this method are not given in the sources.

### 3D Gaussian Splatting
Represents scenes as explicit 3D Gaussians for faster rasterization, trading off the implicit neural volume representation for speed [1795-3d-gaussian-splatting.md]. Details on integration with this method are not given in the sources.

### Neural Radiance Fields (NeRF)
The foundational coordinate-based MLP method for novel view synthesis, which Neural 3D Volume Decomposition extends by adding physical material parameters [4972-neural-radiance-fields.md]. The method explicitly builds on NeRF's two-MLP architecture [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1, unofficial]
*   **Memory Complexity:** Not given in the sources. [S1, unofficial]
*   **Strengths:** Enables high-quality relighting and material-aware view synthesis. Can recover accurate shape and material properties from real-world datasets [S1, unofficial].
*   **Limitations:** Requires images captured under varying illumination conditions. Optimization is complex and may require pre-trained networks for stability [S1, unofficial].
*   **When to use:** When the goal is to recover physical material properties and illumination from a set of images, rather than just novel view synthesis [S1, unofficial].
*   **When not to use:** When only novel view synthesis is needed, or when the input images do not have varying illumination [S1, unofficial].

## Background
Neural 3D Volume Decomposition, also known as Neural-PIL, was introduced in the paper "Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition" [S1, unofficial]. The method addresses the challenge of recovering 3D shape and SVBRDF from images captured under varying lighting conditions, a problem relevant to digitizing real-world objects like statues or landmarks [S1, unofficial]. It builds on the success of NeRF in novel view synthesis and extends it to physical material decomposition [S1, unofficial].

## Sources
[S1] Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition, https://doi.org/10.48550/arxiv.2110.14373, fetched 2026-10-26 (unofficial)
