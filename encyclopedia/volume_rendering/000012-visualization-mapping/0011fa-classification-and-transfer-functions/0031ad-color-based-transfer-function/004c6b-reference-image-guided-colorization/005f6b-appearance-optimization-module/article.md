# Appearance Optimization Module

Also called: AppOpt, Appearance Refinement

## Overview
The Appearance Optimization Module is a learnable neural component that refines 3D Gaussian splat colors by predicting per-image color offsets based on viewing direction and scene context [S1, unofficial]. It adds an adaptive color adjustment to base splat colors, allowing view-dependent appearance modifications [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Splat features (tensor of shape $(N, \text{feature\_dim})$), image ID (identifier for the target view), viewing direction (vector from camera center to splat mean position), and SH degree (spherical harmonics degree for base color) [S1, unofficial].
*   **Outputs:** Refined colors (tensor of shape $(N, 3)$) [S1, unofficial].

**Data Structures:**
*   `embeds`: Learnable parameter tensor of shape $(K, \text{cfg.app\_embed\_dim})$, where $K$ is the number of unique images/views [S1, unofficial].
*   `color_head`: Neural network mapping concatenated input features to a 3-dimensional color offset [S1, unofficial].
*   `feature_dim`: Dimensionality of input splat features; set to 32 if `cfg.app_opt` is enabled, otherwise None [S1, unofficial].

**Parameters:**
*   `cfg.app_embed_dim`: Dimensionality of the per-image embedding vectors [S1, unofficial].
*   `cfg.app_opt_lr`: Base learning rate for appearance optimization [S1, unofficial].
*   `cfg.app_opt_reg`: Weight decay applied to the appearance optimizer [S1, unofficial].
*   `cfg.batch_size`: Batch size used during training, used to scale the learning rate [S1, unofficial].

**Numbered Steps:**
1. Compute color offset by passing splat features, image IDs, viewing directions (means - camera centers), and SH degree to the module [S1, unofficial].
2. Add the module output to the base splat colors: $\text{colors} = \text{colors} + \text{self.splats}[\text{"colors"}]$ [S1, unofficial].
3. Apply sigmoid activation to clamp colors to $[0, 1]$: $\text{colors} = \sigma(\text{colors})$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class AppearanceOptimizationModule(nn.Module):
    def __init__(self, feature_dim, embed_dim):
        super().__init__()
        self.feature_dim = feature_dim
        self.embed_dim = embed_dim
        self.embeds = None  # Initialized at runtime with num_images
        
        # Input dimension combines splat features, embedding, viewing direction, and SH degree
        # Exact SH degree dimension is not given in the sources [G]
        sh_dim = 3  # Assumed standard SH dimension for context [G]
        input_dim = feature_dim + embed_dim + 3 + sh_dim
        self.color_head = nn.Sequential(
            nn.Linear(input_dim, 64),
            nn.ReLU(),
            nn.Linear(64, 3)
        )
        
        # Pitfall: Zero-initialize the last layer to ensure initial offset is zero
        nn.init.zeros_(self.color_head[-1].weight)
        nn.init.zeros_(self.color_head[-1].bias)

    def set_embeddings(self, num_images):
        """Initialize embeddings for all images."""
        self.embeds = nn.Embedding(num_images, self.embed_dim)
        nn.init.zeros_(self.embeds.weight)

    def forward(self, splat_features, image_id, viewing_directions, sh_degree):
        """
        Args:
            splat_features: Tensor of shape (N, feature_dim)
            image_id: Tensor of shape (N,)
            viewing_directions: Tensor of shape (N, 3) representing means - camera centers
            sh_degree: Tensor of shape (N, sh_dim)
        Returns:
            Color offset of shape (N, 3)
        """
        embeddings = self.embeds(image_id)
        combined = torch.cat([splat_features, embeddings, viewing_directions, sh_degree], dim=-1)
        return self.color_head(combined)

def apply_appearance_optimization(colors, app_module, splat_features, image_id, viewing_directions, sh_degree):
    """
    Applies the appearance optimization module to base colors.
    """
    offset = app_module(splat_features, image_id, viewing_directions, sh_degree)
    colors = colors + offset  # Matches source: colors = colors + self.splats["colors"]
    colors = torch.sigmoid(colors)
    return colors

# Optimizer setup per source
app_optimizers = [
    torch.optim.Adam(app_module.embeds.parameters(), lr=cfg.app_opt_lr * math.sqrt(cfg.batch_size) * 10.0, weight_decay=cfg.app_opt_reg),
    torch.optim.Adam(app_module.color_head.parameters(), lr=cfg.app_opt_lr * math.sqrt(cfg.batch_size)),
]
```

**Edge Cases and Pitfalls:**
*   **Initial Bias:** The initial appearance output must be zero to avoid biasing the first render; achieved by zero-initializing the last layer weights and biases of the color head [S1, unofficial].
*   **Learning Rate Scaling:** The actual learning rate for embeddings is scaled by $\sqrt{\text{batch\_size}} \times 10.0$, while the color head is scaled by $\sqrt{\text{batch\_size}}$ [S1, unofficial].
*   **Feature Dimension:** If `cfg.app_opt` is not enabled, `feature_dim` is None, and the module should not be instantiated or called [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The Appearance Optimization Module was introduced as a learnable component for 3D Gaussian Splatting to handle appearance variations [S1, unofficial]. It refines Gaussian splat colors using per-image embeddings and viewing directions, outputting a color offset added to the base splat colors [S1, unofficial].

## Sources
[S1] llazzaroni/AugSplat, https://github.com/llazzaroni/AugSplat, fetched 2026-10-27 (unofficial)
