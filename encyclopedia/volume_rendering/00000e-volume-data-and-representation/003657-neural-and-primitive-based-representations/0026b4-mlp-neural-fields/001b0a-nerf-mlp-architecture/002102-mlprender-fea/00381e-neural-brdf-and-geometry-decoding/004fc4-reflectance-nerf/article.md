# Reflectance NeRF

Also called: Neural Reflectance Model, Reflectance Modification Network

## Overview
Reflectance NeRF is a neural network that predicts per-channel reflectance modification factors for a surface point, conditioned on its 3D position, viewing direction, lighting direction, and base albedo color [S1, unofficial]. It extends the standard NeRF MLP architecture by explicitly modeling view-dependent and light-dependent reflectance properties, allowing for the synthesis of physically plausible appearances under varying illumination [S1, unofficial]. It builds on [Neural BRDF and Geometry Decoding](8139-neural-brdf-and-geometry-decoding.md) by providing a specific implementation for decoding appearance features into reflectance multipliers.

## How it works
**Inputs and Outputs:**
The method takes a batch of surface points and their associated geometric and photometric attributes.
*   **Input:** A tensor of shape $(N, 12)$ containing $N$ samples. The columns are defined as:
    *   Columns 0-2: 3D position $(x, y, z)$ [S1, unofficial].
    *   Columns 3-5: View direction $(x, y, z)$ [S1, unofficial].
    *   Columns 6-8: Light direction $(x, y, z)$ [S1, unofficial].
    *   Columns 9-11: Base color $(R, G, B)$ [S1, unofficial].
*   **Output:** A tensor of shape $(N, 3)$ containing the reflectance modification factors for each color channel, clamped to the range $[0, 1]$.

**Data Structures:**
The core data structure is a multi-layer perceptron (MLP) with 4 hidden layers.
*   **Layer 1:** Linear transformation from 12 to 256 dimensions.
*   **Layers 2-4:** Linear transformations from 256 to 256 dimensions.
*   **Layer 5 (Output):** Linear transformation from 256 to 3 dimensions.
*   **Activation:** ReLU is applied after Layers 1-4. Sigmoid is applied after Layer 5.

**Parameters:**
*   `hidden_dim`: Number of neurons per hidden layer. Default: 256 [S1, unofficial].
*   `learning_rate`: Initial step size for the Adam optimizer. Default: 0.001 (Note: code example uses $10^{-4}$) [S1, unofficial].
*   `num_epochs`: Total number of training passes. Default: 400 (Note: code example uses 100) [S1, unofficial].
*   `batch_size`: Number of samples per gradient update. Default: 1024 [S1, unofficial].
*   `scheduler_patience`: Epochs to wait before reducing learning rate by half. Default: 10 [S1, unofficial].
*   `early_stopping_patience`: Epochs to wait before halting training if validation loss does not improve. Default: 20 [S1, unofficial].

**Algorithm:**
1.  **Normalization:** Normalize the view direction vectors and light direction vectors to unit length using L2 normalization along the last dimension. This ensures that the magnitude of the direction vectors does not incorrectly influence the learned reflectance mapping [S1, unofficial].
    $$ \hat{v} = \frac{v}{\|v\|_2}, \quad \hat{l} = \frac{l}{\|l\|_2} $$
2.  **Concatenation:** Concatenate the position $p$, normalized view direction $\hat{v}$, normalized light direction $\hat{l}$, and base color $c$ into a single 12-dimensional input vector $x$:
    $$ x = [p_x, p_y, p_z, \hat{v}_x, \hat{v}_y, \hat{v}_z, \hat{l}_x, \hat{l}_y, \hat{l}_z, c_R, c_G, c_B] $$
3.  **Feature Extraction:** Pass $x$ through the MLP. Let $W_i, b_i$ be the weights and biases for layer $i$.
    $$ h_1 = \text{ReLU}(W_1 x + b_1) $$
    $$ h_2 = \text{ReLU}(W_2 h_1 + b_2) $$
    $$ h_3 = \text{ReLU}(W_3 h_2 + b_3) $$
    $$ h_4 = \text{ReLU}(W_4 h_3 + b_4) $$
4.  **Reflectance Prediction:** Apply the final linear layer and Sigmoid activation to obtain the reflectance factors $r$:
    $$ r = \sigma(W_5 h_4 + b_5) $$
    where $\sigma(z) = \frac{1}{1 + e^{-z}}$. This clamps the factors to $[0, 1]$ [S1, unofficial].
5.  **Color Computation:** Compute the final rendered color $c_{final}$ by element-wise multiplying the base color channels by the predicted reflectance factors:
    $$ c_{final} = c \odot r $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class ReflectanceNeRF(nn.Module):
    def __init__(self, hidden_dim=256):
        super().__init__()
        # 4 hidden layers of size hidden_dim, input 12, output 3
        self.net = nn.Sequential(
            nn.Linear(12, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, 3),
            nn.Sigmoid()
        )

    def forward(self, positions, view_dirs, light_dirs, base_colors):
        """
        Args:
            positions: (N, 3)
            view_dirs: (N, 3)
            light_dirs: (N, 3)
            base_colors: (N, 3)
        Returns:
            modified_colors: (N, 3)
        """
        # Step 1: Normalize directions
        # [G] Add small epsilon to avoid division by zero if vector is zero
        view_dirs_norm = view_dirs / (torch.norm(view_dirs, dim=-1, keepdim=True) + 1e-8)
        light_dirs_norm = light_dirs / (torch.norm(light_dirs, dim=-1, keepdim=True) + 1e-8)
        
        # Step 2: Concatenate
        # Input shape: (N, 12)
        x = torch.cat([positions, view_dirs_norm, light_dirs_norm, base_colors], dim=-1)
        
        # Step 3 & 4: MLP forward pass
        reflectance_factors = self.net(x)
        
        # Step 5: Compute final color
        modified_colors = base_colors * reflectance_factors
        
        return modified_colors
```

**Edge Cases and Pitfalls:**
*   **Direction Normalization:** View and light directions must be normalized before concatenation. If not, their magnitude will incorrectly influence the learned reflectance mapping [S1, unofficial].
*   **Clipping:** The Sigmoid activation bounds reflectance factors to $[0, 1]$. This prevents negative or excessively bright multipliers but may clip physically plausible values $>1$ [S1, unofficial].

## Subtopics

### [TensoRF Feature Decoding Architecture](8181-tensorf-feature-decoding-architecture.md)
Uses tensor decomposition for feature storage and decoding, offering higher efficiency than dense MLPs. Choose when memory is constrained.

### [Neural Surface Rendering](9960-neural-surface-rendering.md)
Focuses on rendering surfaces defined by implicit fields, often using SDFs. Choose when explicit surface geometry is preferred over volumetric integration.

### [Neural SDF and Material MLPs](10268-neural-sdf-and-material-mlps.md)
General approach to encoding both geometry (SDF) and material properties in MLPs. Choose when joint geometry-material optimization is required.

## Operating characteristics
*   **Time Complexity:** Inference is $O(N \cdot L \cdot D)$, where $N$ is the number of samples, $L$ is the number of layers (4), and $D$ is the hidden dimension (256). Training time depends on the number of epochs and batch size.
*   **Memory Complexity:** The model stores weights for 5 linear layers. Total parameters $\approx 5 \times (12 \times 256 + 256 \times 256 + 256 \times 256 + 256 \times 256 + 256 \times 3) \approx 1.1$ million parameters.
*   **Strengths:** Explicitly models view and light dependence, allowing for dynamic lighting changes without retraining the entire scene. Simple architecture.
*   **Limitations:** The $[0, 1]$ output range limits the dynamic range of reflectance. Requires pre-computed base colors and surface normals/positions.
*   **When to use:** When synthesizing novel views under different lighting conditions for objects with known base albedo.
*   **When not to use:** When high dynamic range reflectance is critical, or when lighting is complex (e.g., environment maps) and not just a single directional light.

## Background
Reflectance NeRF is an extension of the original NeRF architecture, adapting it to handle reflectance properties explicitly. It was introduced in the context of neural rendering for novel view synthesis with variable lighting. The method uses a simple MLP to predict multiplicative factors for base colors, conditioned on geometric and photometric inputs.

## Sources
[S1] matthxie/NeRF-Renderer, https://github.com/matthxie/NeRF-Renderer, fetched 2026-10-27 (unofficial)
