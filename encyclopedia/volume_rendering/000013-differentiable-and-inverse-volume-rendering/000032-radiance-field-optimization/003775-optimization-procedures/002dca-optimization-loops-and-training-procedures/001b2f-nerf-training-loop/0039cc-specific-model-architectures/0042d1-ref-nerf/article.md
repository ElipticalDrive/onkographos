# Ref-NeRF

Also called: Structured View-Dependent Appearance for Neural Radiance Fields

## Overview
Ref-NeRF is a neural radiance field method that replaces the standard parameterization of view-dependent outgoing radiance with a representation of reflected radiance, structured using spatially-varying scene properties to improve the realism of specular reflections [S1]. It addresses the limitation of standard NeRF-based techniques in accurately reproducing glossy surfaces by reparameterizing the directional MLP to accept reflection vectors rather than viewing vectors, and by separating radiance into diffuse and specular components [S1].

## How it works
Ref-NeRF modifies the standard NeRF architecture to explicitly model reflections. The core innovation is the reparameterization of the view-dependent appearance function to be smoother and more interpretable by leveraging geometric normals.

### Inputs and Outputs
*   **Inputs**: 3D position $\mathbf{x}$, viewing direction $\mathbf{d}$, and camera pose information (implicitly via the training data).
*   **Outputs**: Volume density $\sigma$ and color $\mathbf{c}$.

### Data Structures
The model uses two primary MLPs, similar to standard NeRF, but with modified inputs and outputs:
1.  **Spatial MLP**: Takes position $\mathbf{x}$ and outputs a feature vector $\mathbf{f}$ and volume density $\sigma$.
2.  **Directional MLP**: Takes the feature vector $\mathbf{f}$ and a modified direction input to output diffuse color $\mathbf{c}_d$ and specular color $\mathbf{c}_s$.

### Numbered Steps

1.  **Compute Geometry and Normals**:
    The spatial MLP predicts volume density $\sigma(\mathbf{x})$. To obtain the normal vector $\mathbf{n}$, the gradient of the density field is computed:
    $$ \mathbf{n} = \frac{\nabla \sigma(\mathbf{x})}{\|\nabla \sigma(\mathbf{x})\|} $$
    [G] The gradient is typically computed via automatic differentiation during the forward pass.

2.  **Regularize Density for Sharp Normals**:
    Standard NeRF density is "foggy," leading to noisy normals. Ref-NeRF applies a novel regularizer on volume density to concentrate density around surfaces. This regularizer encourages $\sigma$ to be sparse away from surfaces and sharp at boundaries, ensuring accurate normal estimation [S1].

3.  **Reparameterize Directional Input**:
    Instead of passing the viewing direction $\mathbf{d}$ directly to the directional MLP, Ref-NeRF computes the reflection of the viewing vector about the local normal:
    $$ \mathbf{r} = 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n} - \mathbf{d} $$
    The directional MLP takes $\mathbf{r}$ as input. This reparameterization makes the reflected radiance function smoother because it is invariant to surface orientation changes for a fixed lighting environment [S1].

4.  **Integrated Directional Encoding (IDE)**:
    To handle varying materials and textures smoothly, Ref-NeRF applies Integrated Directional Encoding to the reflection vector $\mathbf{r}$. This technique integrates the directional encoding over the cone of the pixel, reducing aliasing and enabling smooth interpolation of the reflected radiance function [S1].

5.  **Separate Diffuse and Specular Components**:
    The directional MLP outputs two color components:
    *   Diffuse color $\mathbf{c}_d$
    *   Specular color $\mathbf{c}_s$
    The final color is the sum:
    $$ \mathbf{c} = \mathbf{c}_d + \mathbf{c}_s $$
    This separation allows the reflected radiance function (driven by $\mathbf{c}_s$) to remain smooth despite variations in material and texture, while $\mathbf{c}_d$ captures view-independent albedo [S1].

6.  **Volume Rendering**:
    The standard volume rendering integral is used to compute the pixel color $\mathbf{C}(\mathbf{r})$ from the sampled $\sigma_i$ and $\mathbf{c}_i$ along the ray [S1, Parent].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.autograd as autograd

class RefNeRF(nn.Module):
    def __init__(self, D=8, W=256, input_ch=3, input_ch_dir=3, 
                 output_ch=4, skips=[4], use_diffuse_specular=True):
        super().__init__()
        self.D = D
        self.W = W
        self.input_ch = input_ch
        self.input_ch_dir = input_ch_dir
        self.use_diffuse_specular = use_diffuse_specular
        
        # Spatial MLP
        self.spatial_mlp = nn.Sequential(
            nn.Linear(input_ch, W),
            nn.ReLU(),
            *[nn.Sequential(nn.Linear(W, W), nn.ReLU()) for _ in range(D-1)],
            nn.Linear(W, output_ch) # Outputs sigma and features
        )
        
        # Directional MLP
        # Input: features from spatial MLP + encoded reflection direction
        # Output: diffuse and specular colors
        dir_input_ch = W + input_ch_dir # Assuming features are concatenated or added
        # Note: Standard NeRF uses concatenation. Ref-NeRF uses reflection vector.
        self.dir_mlp = nn.Sequential(
            nn.Linear(dir_input_ch, W),
            nn.ReLU(),
            *[nn.Sequential(nn.Linear(W, W), nn.ReLU()) for _ in range(D//2)],
            nn.Linear(W, 3) # Diffuse color
        )
        self.specular_mlp = nn.Sequential(
            nn.Linear(dir_input_ch, W),
            nn.ReLU(),
            *[nn.Sequential(nn.Linear(W, W), nn.ReLU()) for _ in range(D//2)],
            nn.Linear(W, 3) # Specular color
        )

    def compute_normal(self, sigma):
        """Compute normal from density gradient."""
        sigma.requires_grad_(True)
        grad_outputs = torch.ones_like(sigma)
        gradients = autograd.grad(
            outputs=sigma,
            inputs=sigma, # In practice, this is d(sigma)/d(x)
            grad_outputs=grad_outputs,
            create_graph=True,
            retain_graph=True,
            only_inputs=True
        )[0]
        # Normalize
        normals = gradients / (gradients.norm(dim=-1, keepdim=True) + 1e-6)
        return normals

    def forward(self, x, d):
        """
        x: [N, 3] positions
        d: [N, 3] viewing directions
        """
        # 1. Spatial MLP
        sigma_and_features = self.spatial_mlp(x)
        sigma = sigma_and_features[..., :1]
        features = sigma_and_features[..., 1:]
        
        # 2. Compute Normal
        # Note: In training, we need gradients w.r.t x. 
        # The spatial_mlp output sigma must be differentiated w.r.t x.
        # This is typically handled by the renderer's ray marching loop 
        # where sigma is computed for each sample point.
        # For a single point query:
        x.requires_grad_(True)
        sigma_val = self.spatial_mlp(x)[..., :1]
        normals = self.compute_normal(sigma_val)
        
        # 3. Reparameterize Direction
        # r = 2(d . n)n - d
        dot_product = torch.sum(d * normals, dim=-1, keepdim=True)
        r = 2 * dot_product * normals - d
        
        # 4. Integrated Directional Encoding (IDE)
        # IDE involves integrating the positional encoding over the cone.
        # For implementation, this replaces the standard positional encoding 
        # of the direction vector.
        # r_encoded = integrated_directional_encoding(r, ...)
        # For simplicity, we assume a placeholder for the IDE function 
        # which maps r to a higher-dimensional space.
        r_encoded = r # Placeholder: Replace with actual IDE implementation
        
        # 5. Directional MLPs
        dir_input = torch.cat([features, r_encoded], dim=-1)
        c_diffuse = self.dir_mlp(dir_input)
        c_specular = self.specular_mlp(dir_input)
        
        # 6. Combine Colors
        c = c_diffuse + c_specular
        
        return sigma, c
```

### Parameters and Typical Values
*   **PSNR**: 35.6 dB on the evaluated dataset.
*   **Mean Angular Error (MAE)**: 11.5 degrees for normals.
*   **Comparison**: Outperforms mip-NeRF (30.3 dB PSNR, 59.5 degrees MAE) [S1].

### Edge Cases and Pitfalls
*   **Foggy Geometry**: Standard NeRF geometry is not tightly concentrated at surfaces, leading to noisy normals. This is mitigated by the volume density regularizer [S1].
*   **Noisy Normals**: Without regularization, normal vectors are too noisy for accurate reflection computation [S1].

## Operating characteristics
*   **Performance**: Ref-NeRF achieves 35.6 dB PSNR and 11.5 degrees MAE for normals on the evaluated dataset, significantly outperforming mip-NeRF (30.3 dB PSNR, 59.5 degrees MAE) [S1].
*   **Strengths**: Improves realism and accuracy of specular reflections; provides an interpretable internal representation useful for scene editing [S1].
*   **Limitations**: Relies on accurate normal estimation, which requires the density regularizer. Computationally more complex due to gradient computation for normals and IDE [S1].

## Background
Ref-NeRF was introduced by Dor Verbin, Peter Hedman, Ben Mildenhall, Todd Zickler, Jonathan T. Barron, and Pratul P. Srinivasan from Harvard University and Google Research [S1]. It was published as arXiv:2112.03907v1 on Dec 7, 2021 [S1]. The method addresses the failure of NeRF-based techniques to accurately capture glossy surfaces by structuring the view-dependent appearance function [S1].

## Sources
[S1] Ref-NeRF: Structured View-Dependent Appearance for Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00541, fetched 2026-10-26
