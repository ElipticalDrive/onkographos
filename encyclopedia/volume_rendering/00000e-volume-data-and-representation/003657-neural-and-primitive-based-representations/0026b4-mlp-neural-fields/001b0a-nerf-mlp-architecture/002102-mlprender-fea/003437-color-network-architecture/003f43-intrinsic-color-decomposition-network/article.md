# Intrinsic Color Decomposition Network

Also called: Color Decomposition, Intrinsic Color Decomposition

## Overview
The Intrinsic Color Decomposition Network disentangles color-shape ambiguities by predicting separate albedo (view-invariant) and shading (view-dependent) components, which are summed to produce the final color [S1, S2]. Unlike standard NeRF architectures that use a single view-dependent MLP for color, this method improves extrapolation performance in novel view synthesis by allowing the view-independent factor to provide reasonable predictions when facing large viewing direction shifts [S2].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Point location $x \in \mathbb{R}^3$.
    *   Features $f$ extracted from the SDF MLP or density grid at location $x$ [S1, S2].
    *   Viewing direction $d \in \mathbb{R}^3$ (normalized) [S2].
    *   Reflection direction $r$ (used in the shading branch of the Neuralangelo variant) [S1].
*   **Outputs:**
    *   Final color $C \in [0, 1]^3$ [S1].
    *   Intermediate view-dependent color $c_{vd} \in \mathbb{R}^3$ [S2].
    *   Intermediate view-independent color $c_{vi} \in \mathbb{R}^3$ [S2].
    *   Shading scalar $C_s \in \mathbb{R}$ (Neuralangelo variant) [S1].

### Data Structures
The network consists of two parallel Multi-Layer Perceptrons (MLPs):
1.  **Albedo/View-Independent Branch ($M_{rgb\_vi}$):**
    *   Input: Features $f$.
    *   Output: $c_{vi} \in \mathbb{R}^3$ (RGB) [S2] or $C_a \in \mathbb{R}^3$ (RGB) [S1].
    *   Architecture: Two hidden layers [S1].
2.  **Shading/View-Dependent Branch ($M_{rgb\_vd}$):**
    *   Input: Features $f$ and viewing direction $d$ [S2]. In the Neuralangelo variant, it also takes reflection direction and view direction [S1].
    *   Output: $c_{vd} \in \mathbb{R}^3$ (RGB) [S2] or $C_s \in \mathbb{R}$ (Gray/Scalar) [S1].
    *   Architecture: Two hidden layers [S1].

### Steps and Formulas

1.  **Compute View-Independent Color:**
    Pass the features $f$ through the view-independent MLP to obtain the albedo or diffuse color component.
    $$c_{vi} = M_{rgb\_vi}(f) \quad \text{or} \quad C_a = M_{albedo}(f) \quad [S1, S2]$$

2.  **Compute View-Dependent Color:**
    Pass the features $f$ and viewing direction $d$ through the view-dependent MLP.
    $$c_{vd} = M_{rgb\_vd}(f, d) \quad [S2]$$
    *Note: In the Neuralangelo implementation, the shading branch predicts a single scalar $C_s$ representing intensity changes (specular highlights, shadows, exposure), while the albedo branch predicts the full RGB vector [S1].*

3.  **Combine Components:**
    Sum the view-dependent and view-independent factors to get the raw color prediction.
    $$c = c_{vd} + c_{vi} \quad [S2]$$
    $$C_{raw} = C_a + C_s \quad [S1]$$

4.  **Normalize Output:**
    Apply a Sigmoid function to normalize the final color into the range $[0, 1]$.
    $$C = \Phi(C_{raw}) = \frac{1}{1 + e^{-C_{raw}}} \quad [S1]$$
    *Note: The Lightning NeRF variant [S2] does not explicitly mention a Sigmoid activation in the final combination step, assuming the MLP outputs are bounded or the loss handles normalization. However, standard practice for color output is clamping or sigmoid. We follow [S1] for the explicit normalization formula.*

5.  **Regularization (Training):**
    During training, apply an L1 regularizer to the view-dependent color to prevent it from dominating or becoming unstable.
    $$L_r = \frac{1}{M} \sum_{j=1}^{M} \|c_{vd,j}\|_1 \quad [S2]$$
    Where $M$ is the number of view-dependent color samples.

6.  **Total Loss:**
    Combine the photometric loss $L_p$ (weighted MSE between predicted and ground truth color) with the regularization loss $L_r$ using weight $\lambda$.
    $$L = L_p + \lambda L_r \quad [S2]$$
    $$L_p = \frac{1}{N} \sum_{i=1}^{N} \langle w_i \rangle \cdot \|\hat{C}(r_i) - C(r_i)\|_2^2 \quad [S2]$$
    Where $\langle \cdot \rangle$ denotes the stop-gradient operation [S2].

### Parameters
*   **Regularization Weight ($\lambda$):** Set to $0.01$ [S2].
*   **Hidden Layers:** Both branches use two hidden layers [S1].
*   **Optimizer:** Adam for MLP networks with initial learning rate $0.01$ [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class IntrinsicColorDecompositionNetwork(nn.Module):
    """
    Implements Intrinsic Color Decomposition.
    Supports two variants based on source [S1] (Neuralangelo) and [S2] (Lightning NeRF).
    """
    def __init__(self, feature_dim, use_scalar_shading=True):
        super().__init__()
        self.use_scalar_shading = use_scalar_shading
        
        # View-Independent Branch (Albedo)
        # Input: feature_dim, Output: 3 (RGB)
        self.albedo_mlp = nn.Sequential(
            nn.Linear(feature_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 3)
        )
        
        # View-Dependent Branch (Shading)
        if use_scalar_shading:
            # Neuralangelo variant: Output 1 scalar (intensity)
            # Input: feature_dim + 3 (view_dir) + 3 (reflection_dir)
            self.shading_mlp = nn.Sequential(
                nn.Linear(feature_dim + 6, 256),
                nn.ReLU(),
                nn.Linear(256, 256),
                nn.ReLU(),
                nn.Linear(256, 1)
            )
        else:
            # Lightning NeRF variant: Output 3 (RGB)
            # Input: feature_dim + 3 (view_dir)
            self.shading_mlp = nn.Sequential(
                nn.Linear(feature_dim + 3, 256),
                nn.ReLU(),
                nn.Linear(256, 256),
                nn.ReLU(),
                nn.Linear(256, 3)
            )

    def forward(self, features, view_dir, reflection_dir=None):
        """
        Args:
            features: Tensor of shape (N, feature_dim) from SDF/NeRF encoder
            view_dir: Tensor of shape (N, 3), normalized viewing direction
            reflection_dir: Tensor of shape (N, 3), optional, for Neuralangelo variant
        
        Returns:
            color: Tensor of shape (N, 3), final color in [0, 1]
            c_vd: Tensor of shape (N, 3) or (N, 1), view-dependent component
            c_vi: Tensor of shape (N, 3), view-independent component
        """
        # 1. Compute View-Independent Color (Albedo)
        c_vi = self.albedo_mlp(features) # Shape: (N, 3)
        
        # 2. Compute View-Dependent Color (Shading)
        if self.use_scalar_shading:
            if reflection_dir is None:
                raise ValueError("Reflection direction required for scalar shading variant")
            # Concatenate features, view_dir, and reflection_dir
            shading_input = torch.cat([features, view_dir, reflection_dir], dim=-1)
            c_s = self.shading_mlp(shading_input) # Shape: (N, 1)
            
            # In Neuralangelo, shading is scalar intensity applied to albedo?
            # Source [S1] says: C = Phi(Ca + Cs). Ca is RGB, Cs is Gray.
            # This implies Cs is broadcasted or added to each channel.
            # Usually shading modulates intensity. Let's assume element-wise addition
            # where Cs is broadcast to 3 channels.
            c_vd = c_s.repeat(1, 3) # Shape: (N, 3)
        else:
            # Lightning NeRF variant
            shading_input = torch.cat([features, view_dir], dim=-1)
            c_vd = self.shading_mlp(shading_input) # Shape: (N, 3)
            
        # 3. Combine
        c_raw = c_vi + c_vd
        
        # 4. Normalize
        color = torch.sigmoid(c_raw)
        
        return color, c_vd, c_vi

def compute_loss(color_pred, color_gt, c_vd, lambda_reg=0.01):
    """
    Computes the total loss including photometric loss and L1 regularization on c_vd.
    """
    # Photometric Loss (MSE)
    # Source [S2] mentions weighted loss with stop-gradient, but standard MSE is:
    l_p = F.mse_loss(color_pred, color_gt)
    
    # Regularization Loss (L1 on view-dependent color)
    # Source [S2]: L_r = (1/M) * sum ||c_vd||_1
    l_r = torch.mean(torch.abs(c_vd))
    
    total_loss = l_p + lambda_reg * l_r
    return total_loss
```

### Edge Cases and Pitfalls
*   **Flat Surface Carving:** Flat surfaces tend to be "carved away" (reconstructed as non-flat) when using this decomposition. This is potentially due to instability in the dot product computation for reflection direction in the shading branch [S1].
*   **Extrapolation Failure without Decomposition:** Without color decomposition, the model fails in novel view synthesis in extrapolation settings (e.g., shifting viewing direction 2 meters left from training views) because the single view-dependent MLP is not optimized for directions outside the training domain [S2].
*   **Shading Branch Capacity:** The single-channel design for the shading branch (in Neuralangelo) limits its capacity, which encourages the albedo branch to learn the view-invariant color better [S1].

## Subtopics

### Neural Textures Architecture
*   **What it is:** A method that stores color information in a 2D texture map and uses an MLP to predict texture coordinates from 3D positions and viewing directions.
*   **Difference:** Unlike Intrinsic Color Decomposition which splits color into albedo/shading MLPs, Neural Textures relies on explicit 2D texture storage and coordinate prediction.
*   **When to choose:** When explicit texture maps are preferred over implicit neural representations for color.
*   **Link:** [Neural Textures Architecture](9534-neural-textures-architecture.md)

### Decomposition Network
*   **What it is:** A general term for networks that decompose scene properties (like depth, normals, or color) into separate components.
*   **Difference:** Intrinsic Color Decomposition is a specific instance focused on albedo/shading separation for NeRFs. Other decomposition networks might target different attributes or use different architectures.
*   **When to choose:** When the specific albedo/shading split is required for intrinsic image analysis or improved extrapolation.
*   **Link:** [Decomposition Network](12532-decomposition-network.md)

### Squeeze-and-Excitation Module
*   **What it is:** An attention mechanism that explicitly models interdependencies between channels in convolutional neural networks.
*   **Difference:** SE Modules are used for channel-wise recalibration within a network, whereas Intrinsic Color Decomposition splits the network into two parallel branches for semantic color components.
*   **When to choose:** When channel attention is needed to improve feature representation, not for intrinsic color separation.
*   **Link:** [Squeeze-and-Excitation Module](14050-squeeze-and-excitation-module.md)

## Operating characteristics

*   **Time Complexity:** The method adds computational overhead compared to a single color MLP, as it requires two separate MLP passes (albedo and shading) and their combination. However, the branches are described as "lightweight" with only two hidden layers [S1, S2].
*   **Memory Complexity:** Increases memory usage by storing parameters for two MLPs instead of one. The feature dimension $f$ is shared, so no additional feature storage is needed beyond the base NeRF/SDF representation.
*   **Measured Performance:**
    *   On KITTI-360, the method (Lightning NeRF) achieved a PSNR of 23.71 and SSIM of 0.8521, outperforming NeRF-W (22.77 PSNR) and TensoRF (23.55 PSNR) [S2].
    *   On Argoverse2, it achieved 30.62 PSNR and 0.8559 SSIM, outperforming Tetra-NeRF (29.59 PSNR) [S2].
*   **Strengths:**
    *   Improves novel view synthesis in extrapolation settings where viewing directions shift significantly from training data [S2].
    *   Better simulates the physical world by separating diffuse (view-independent) and specular (view-dependent) components [S2].
*   **Limitations:**
    *   May cause flat surfaces to be carved away due to instability in reflection dot products [S1].
    *   Neuralangelo authors did not observe improvements in surface qualities using this decomposition design [S1].
*   **When to use:** When extrapolation to novel views is critical (e.g., autonomous driving scenarios with limited scan coverage) [S2].
*   **When not to use:** When surface reconstruction quality of flat planes is the primary concern, or when computational resources are extremely limited [S1].

## Background

*   **Origin:** The concept of decomposing color into albedo and shading is rooted in intrinsic image decomposition [S1, S2]. The specific application to NeRF architectures was introduced in **Neuralangelo** (CVPR 2023) [S1] and **Lightning NeRF** (ICRA 2024) [S2].
*   **History:** Standard NeRF uses a single view-dependent MLP for color, which simplifies the physical world but fails in extrapolation [S2]. Neuralangelo attempted to disentangle color-shape ambiguities by implementing explicit albedo and shading branches [S1]. Lightning NeRF further refined this for autonomous driving, using lightweight MLPs for view-dependent and view-independent factors to handle large viewing direction shifts [S2].
*   **Usage:** Used in high-fidelity neural surface reconstruction (Neuralangelo) and efficient hybrid scene representation for autonomous driving (Lightning NeRF) [S1, S2].

## Sources

[S1] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S2] Lightning NeRF: Efficient Hybrid Scene Representation for Autonomous Driving, https://doi.org/10.1109/icra57147.2024.10611130, fetched 2026-10-27
