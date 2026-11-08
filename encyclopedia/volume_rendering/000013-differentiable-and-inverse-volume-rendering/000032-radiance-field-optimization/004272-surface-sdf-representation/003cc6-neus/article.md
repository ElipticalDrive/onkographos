# NeuS

Also called: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction, NeuS Volume Rendering with Softplus SDF, NeuS, NeuS2 Static Reconstruction, Neural Implicit Surface, Neural Surface, Neural Surface Reconstruction

## Overview
NeuS is a neural surface reconstruction method that represents a surface as the zero-level set of a signed distance function (SDF) and trains it using a novel volume rendering scheme to avoid geometric bias inherent in standard volume rendering [S1, unofficial]. It differs from its sibling [Surface NeRF](7385-surface-nerf.md) by explicitly modeling the scene with an SDF and a color field, using a specific S-density formulation to derive unbiased alpha values, whereas Surface NeRF typically relies on a differentiable surface rendering approach or different density-to-alpha mappings [S1, unofficial][S2]. It solves the problem of reconstructing high-quality, watertight meshes from multi-view images by combining the robustness of volume rendering with the accuracy of implicit surface representations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of posed RGB images $\{I_k\}$ of a 3D object.
*   **Outputs:** A trained SDF network $f(\mathbf{x})$ and a color network $c(\mathbf{x}, \mathbf{v})$, from which a mesh is extracted via Marching Cubes [S1, unofficial][S2].

**Data Structures:**
The scene is represented by two Multi-Layer Perceptrons (MLPs) [S1, unofficial]:
1.  **SDF Network ($f$):** Maps $\mathbf{x} \in \mathbb{R}^3$ to a scalar signed distance $d$.
    *   Architecture: 8 hidden layers, hidden size 256.
    *   Activation: Softplus with $\beta=100$ [S1, unofficial].
    *   Skip connection: Input concatenated to the output of the 4th layer [S1, unofficial].
    *   Output: Tuple $(d, \mathbf{g})$ where $d$ is the SDF value and $\mathbf{g} \in \mathbb{R}^{256}$ is a geometry feature vector [S1, unofficial][S3].
2.  **Color Network ($c$):** Maps $(\mathbf{x}, \mathbf{v})$ to RGB color.
    *   Architecture: 4 hidden layers, hidden size 256.
    *   Inputs: Spatial location $\mathbf{p}$, view direction $\mathbf{v}$, normal vector $\mathbf{n} = \nabla f(\mathbf{p})$, and the 256-dimensional feature vector $\mathbf{g}$ from the SDF network [S1, unofficial].
    *   Positional Encoding: Applied to $\mathbf{p}$ (6 frequencies) and $\mathbf{v}$ (4 frequencies) [S1, unofficial].
    *   Weight normalization is used to stabilize training [S1, unofficial].

**Rendering Procedure:**
A ray is parameterized as $\mathbf{p}(t) = \mathbf{o} + t\mathbf{v}$, where $\mathbf{o}$ is the camera center and $\mathbf{v}$ is the unit direction vector [S1, unofficial]. The rendered color $\hat{C}(\mathbf{o}, \mathbf{v})$ is computed via volume rendering [S1, unofficial]:

$$
\hat{C}(\mathbf{o}, \mathbf{v}) = \sum_{i=1}^N w_i \mathbf{c}_i
$$

where $\mathbf{c}_i$ is the color at the $i$-th sample point and $w_i$ is the weight. The weight is derived from the **S-density** $\phi_s(f(\mathbf{x}))$, which is the probability density function induced by the SDF [S1, unofficial].

**S-Density and Alpha Calculation:**
The S-density is defined as the derivative of the Sigmoid function $\Phi_s(x) = (1 + e^{-sx})^{-1}$ [S1, unofficial]:

$$
\phi_s(x) = \frac{s e^{-sx}}{(1 + e^{-sx})^2}
$$

Here, $s$ is a trainable parameter controlling the sharpness of the distribution; the standard deviation is $1/s$ [S1, unofficial]. As training converges, $s$ increases, and the distribution approaches a Dirac delta function at the surface [S1, unofficial].

To compute the alpha value $\alpha_i$ for the segment between section points $q_i$ and $q_{i+1}$, NeuS uses the cumulative distribution function $\Phi_s$ [S1, unofficial]:

$$
\alpha_i = \max\left( \frac{\Phi_s(f(q_i)) - \Phi_s(f(q_{i+1}))}{\Phi_s(f(q_i))}, 0 \right)
$$

The color $\mathbf{c}_i$ is evaluated at the midpoint $p_i = \mathbf{o} + \frac{t_i + t_{i+1}}{2}\mathbf{v}$ [S1, unofficial].

**Sampling Strategy:**
1.  **Uniform Sampling:** 64 points are sampled uniformly along the ray [S1, unofficial].
2.  **Importance Sampling:** Iterated $k=4$ times. In the $i$-th iteration, a fixed $s$ value of $32 \times 2^i$ is used for coarse probability estimation. 16 additional points are sampled per iteration [S1, unofficial].
3.  **Total Points:** 128 points per ray [S1, unofficial].
4.  **Background:** For settings without foreground masks, 32 extra points are sampled outside the bounding sphere, representing the background using a NeRF++ model [S1, unofficial].

**Training and Loss:**
The model is trained using the ADAM optimizer [S1, unofficial]. The loss function combines color reconstruction and an Eikonal regularization term [S3]:

$$
\mathcal{L} = \mathcal{L}_{\text{color}} + \beta \mathcal{L}_{\text{eikonal}}
$$

*   $\mathcal{L}_{\text{color}}$: Huber loss between rendered and ground truth pixels [S3].
*   $\mathcal{L}_{\text{eikonal}}$: Regularizes the gradient magnitude of the SDF to 1: $\frac{1}{mn} \sum_{k,i} (\|\mathbf{n}_{k,i}\| - 1)^2$, where $\mathbf{n}_{k,i} = \nabla f(\mathbf{p}_{k,i})$ [S3].
*   $\beta$: Weight for the Eikonal loss [S3].

**Optimization Schedule:**
*   Learning rate warmup: Linear from 0 to $5 \times 10^{-4}$ over the first 5,000 iterations [S1, unofficial].
*   Decay: Cosine decay to a minimum of $2.5 \times 10^{-5}$ [S1, unofficial].
*   Total iterations: 300,000 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SDFNetwork(nn.Module):
    def __init__(self, hidden_dim=256):
        super().__init__()
        self.layers = nn.ModuleList([nn.Linear(hidden_dim, hidden_dim) for _ in range(8)])
        self.input_layer = nn.Linear(3, hidden_dim)
        self.output_layer = nn.Linear(hidden_dim, 1) # SDF value
        self.feature_layer = nn.Linear(hidden_dim, hidden_dim) # Feature vector
        
    def forward(self, x):
        h = self.input_layer(x)
        for i, layer in enumerate(self.layers):
            if i == 4:
                h = torch.cat([h, x], dim=-1) # Skip connection
            h = F.softplus(layer(h), beta=100)
        sdf = self.output_layer(h)
        feature = self.feature_layer(h)
        return sdf, feature

class ColorNetwork(nn.Module):
    def __init__(self, hidden_dim=256):
        super().__init__()
        self.layers = nn.ModuleList([nn.Linear(hidden_dim, hidden_dim) for _ in range(4)])
        self.output_layer = nn.Linear(hidden_dim, 3)
        
    def forward(self, x, v, normal, feature):
        # Concatenate inputs: x, v, normal, feature
        # Note: Positional encoding should be applied to x and v before this call
        input_vec = torch.cat([x, v, normal, feature], dim=-1)
        h = input_vec
        for layer in self.layers:
            h = F.relu(layer(h))
        return self.output_layer(h)

def sigmoid_cdf(x, s):
    return 1.0 / (1.0 + torch.exp(-s * x))

def compute_alpha(sdf_vals, s):
    """
    sdf_vals: tensor of shape (N,) containing f(q_i)
    s: scalar sharpness parameter
    Returns alpha tensor of shape (N-1,)
    """
    cdf_prev = sigmoid_cdf(sdf_vals[:-1], s)
    cdf_next = sigmoid_cdf(sdf_vals[1:], s)
    # Avoid division by zero
    alpha = (cdf_prev - cdf_next) / (cdf_prev + 1e-5)
    alpha = torch.clamp(alpha, min=0.0)
    return alpha

def volume_render(sdf_net, color_net, rays_o, rays_d, s, num_samples=128):
    """
    rays_o: (H, W, 3)
    rays_d: (H, W, 3)
    Returns: rendered_color (H, W, 3), alpha (H, W, N)
    """
    # 1. Sample points
    # Uniform sampling
    t_uniform = torch.linspace(0, 1, 64).to(rays_o.device)
    t_samples = t_uniform.expand(rays_o.shape[0], rays_o.shape[1], 64)
    
    # Importance sampling (simplified: just add points based on s)
    # In practice, this involves iterative resampling. 
    # Here we assume final t_samples contains 128 points.
    
    # 2. Evaluate SDF
    # p = o + t * d
    positions = rays_o.unsqueeze(-1) + t_samples.unsqueeze(-1) * rays_d.unsqueeze(-1) # (H, W, N, 3)
    sdf_vals, features = sdf_net(positions.reshape(-1, 3))
    sdf_vals = sdf_vals.reshape(rays_o.shape[0], rays_o.shape[1], -1)
    features = features.reshape(rays_o.shape[0], rays_o.shape[1], -1)
    
    # 3. Compute Alphas
    alphas = compute_alpha(sdf_vals, s) # (H, W, N-1)
    
    # 4. Compute Transmittance and Weights
    # T_i = product(1 - alpha_j) for j < i
    # Weights w_i = alpha_i * T_i
    # Note: Standard volume rendering accumulation
    T = torch.cumprod(1 - alphas, dim=-1)
    T = torch.cat([torch.ones_like(T[..., :1]), T], dim=-1)
    weights = alphas * T[..., :-1]
    
    # 5. Compute Colors
    # Midpoints for color evaluation
    midpoints = (positions[..., :-1, :] + positions[..., 1:, :]) / 2.0
    normals = torch.autograd.grad(sdf_vals.sum(), positions, create_graph=True)[0]
    normals = normals[..., :-1, :] # Align with midpoints/weights
    
    # Normalize normals
    normals = normals / (torch.norm(normals, dim=-1, keepdim=True) + 1e-5)
    
    # Get colors
    # Note: Positional encoding for midpoints and view directions is required
    colors = color_net(midpoints, rays_d, normals, features[..., :-1, :])
    
    # 6. Accumulate
    rendered_color = torch.sum(weights.unsqueeze(-1) * colors, dim=-2)
    
    return rendered_color, alphas

def train_step(sdf_net, color_net, rays_o, rays_d, gt_images, optimizer):
    optimizer.zero_grad()
    
    # Initial s value
    s = 32.0 
    
    # Render
    pred_color, _ = volume_render(sdf_net, color_net, rays_o, rays_d, s)
    
    # Color Loss (Huber)
    loss_color = F.huber_loss(pred_color, gt_images)
    
    # Eikonal Loss
    # Requires gradient of SDF w.r.t x
    # This is typically computed during the forward pass or via a separate call
    # For simplicity, assume we have normals from volume_render or recompute
    # L_eikonal = mean((||grad f(x)|| - 1)^2)
    # ... implementation details omitted for brevity ...
    loss_eikonal = 0.0 
    
    loss = loss_color + 0.1 * loss_eikonal # beta = 0.1
    loss.backward()
    optimizer.step()
    
    return loss.item()
```

## Subtopics

### [HAME-NeRF](9325-hame-nerf.md)
HAME-NeRF is a two-stage framework for extracting high-fidelity, textured surface meshes from RGB images by defining volume density as a Secant Hyperbolic Function applied to a Signed Distance Function (SDF). It builds on [NeuS](6060-neus.md) by introducing a spatially-varying sharpness parameter $\beta(\mathbf{x})$ learned via an MLP, which enhances flexibility for complex surfaces compared to methods using fixed $\beta$ values. Unlike [NeuS](6060-neus.md) which focuses on differentiable volume rendering for surface representation, HAME-NeRF explicitly optimizes a mesh through differentiable rendering and iterative refinement, achieving superior geometric accuracy and visual quality on synthetic benchmarks compared to [NeRF2Mesh](52-nerf2mesh.md).

### [D-NeuS](10217-d-neus.md)
D-NeuS is a volume rendering-based neural implicit surface reconstruction method that recovers fine-level geometric details by extending NeuS with two additional loss functions: a geometry bias loss and a multi-view feature consistency loss. It addresses the ambiguity in geometry and appearance representation inherent in volume rendering by ensuring consistency between the rendered surface points and the underlying signed distance field (SDF) zero-crossings. Unlike NeuS, which may suffer from geometry bias due to the transformation from SDF to density, D-NeuS explicitly penalizes this bias and enforces multi-view feature consistency on surface points derived via linear interpolation of SDF zero-crossings, rather than using patch-based photometric consistency on all sampled points.

### [NeuS2](10220-neus2.md)
NeuS2 is a fast neural surface reconstruction approach for multi-view RGB input that achieves two orders of magnitude improvement in training speed over NeuS without compromising reconstruction quality, applicable to both static and dynamic scenes. It sets itself apart by parameterizing the neural network-encoded Signed Distance Field (SDF) using multi-resolution hash tables of learnable feature vectors, as introduced in Instant-NGP, and by calculating second-order derivatives using a novel lightweight formula tailored to ReLU-based MLPs to leverage CUDA parallelism and reduce memory footprint.

## Operating characteristics
*   **Time Complexity:** Training is slow. Rendering a 1600x1200 image takes ~320 seconds on an Nvidia 2080Ti GPU in the default volume rendering setting [S1, unofficial]. With sphere tracing acceleration, this drops to ~60 seconds [S1, unofficial]. On an A100 GPU at 1080p, it renders at 0.8 +- 0.2 FPS [S2]. It is slower than standard NeRF because SDF queries require more network evaluations per ray [S2].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** On the NeRF Synthetic (Blender) dataset, NeuS achieved PSNR = 28.4 +- 0.8 dB [S2]. On the DTU dataset, it achieved the most accurate surface reconstruction with Chamfer distance = 0.62 +- 0.08 mm [S2].
*   **Strengths:** Robust optimization for complex structures; handles severe occlusions; accurate surface extraction via zero-level set; no geometric bias in first-order approximation [S1, unofficial].
*   **Limitations:** Slow training and rendering; fails on textureless regions (e.g., metal surfaces) due to ambiguity; random initialization causes axis-aligned artifacts due to spectral bias of positional encoding [S1, unofficial].

## Background
NeuS was introduced in the paper "NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction" [S1, unofficial]. It addresses the difficulty of extracting high-quality surfaces from neural radiance fields by combining the accuracy of neural SDF models with the robustness of volume rendering. It was developed to overcome the inherent geometric bias present when applying standard volume rendering to SDF-associated densities [S1, unofficial].

## Sources
[S1] NeuS: Learning Neural Implicit Surfaces by Volume Rendering for Multi-view Reconstruction, https://doi.org/10.48550/arxiv.2106.10689, fetched 2026-10-26 (unofficial)
[S2] Neural Rendering Techniques, https://doi.org/10.5281/zenodo.19614729, fetched 2026-10-26
[S3] NeuS2: Fast Learning of Neural Implicit Surfaces for Multi-view Reconstruction, https://doi.org/10.1109/iccv51070.2023.00305, fetched 2026-10-26
[S4] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S5] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
