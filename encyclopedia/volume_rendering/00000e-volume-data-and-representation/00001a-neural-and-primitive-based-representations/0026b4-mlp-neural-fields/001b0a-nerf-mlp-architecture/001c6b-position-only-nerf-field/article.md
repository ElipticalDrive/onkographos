# Position-Only NeRF Field

Also called: NeRF Field (Position-Only MLP), Neural Radiance Field, Position-only MLP, coordinate MLP with skip connections

## Overview
A position-only NeRF field is an implicit volume representation that maps a 3D spatial coordinate directly to volume density and emitted color, omitting the viewing direction input to reduce model size and training time [S1, unofficial][S2]. Unlike the full NeRF architecture which models view-dependent effects, this variant assumes isotropic appearance, making it suitable for scenes where lighting is uniform or view dependence is negligible [S1, unofficial]. It serves as a simplified baseline for neural radiance fields, trading view-dependent accuracy for computational efficiency [S1, unofficial].

## How it works
The method uses a Multi-Layer Perceptron (MLP) with Fourier positional encoding and skip connections to regress density and color from 3D points [S1, unofficial][S2].

### Inputs and Outputs
- **Input**: A batch of 3D points with shape `(*batch, 3)` [S1, unofficial].
- **Output**: A tuple `(density, color)` where `density` has shape `(*batch, 1)` and `color` has shape `(*batch, 3)` [S1, unofficial].

### Data Structures
The core data structure is the MLP itself, defined by the following hyperparameters:
- `num_octaves`: Number of octaves for positional encoding. Default: 10 [S1, unofficial].
- `hidden_features`: Width of the hidden layers. Default: 256 [S1, unofficial].
- `hidden_layers`: Number of hidden linear layers. Default: 4 [S1, unofficial].
- `skip_layer`: Index of the layer where the original encoded position is concatenated. Default: 2 [S1, unofficial].

### Steps
1. **Positional Encoding**: Apply Fourier positional encoding to the input 3D points. This maps each coordinate $x \in \mathbb{R}^3$ to a higher-dimensional vector using sine and cosine functions at multiple frequencies [S1, unofficial][S2]. The number of frequencies is determined by `num_octaves` [S1, unofficial].
2. **MLP Forward Pass**: Pass the encoded points through the MLP [S1, unofficial].
3. **Skip Connections**: For each hidden layer $i$, if $i == \text{skip\_layer}$, concatenate the current layer's input (the original encoded position) with the layer's output along the feature dimension [S1, unofficial].
4. **Activation**: Apply ReLU activation to the output of every linear layer except the final heads [S1, unofficial].
5. **Density Head**: Pass the final hidden representation through a linear layer mapping `hidden_features` to 1. Apply the softplus activation function to ensure non-negative density [S1, unofficial][S2].
6. **Color Head**: Pass the final hidden representation through a linear layer mapping `hidden_features` to 3. Apply the sigmoid activation function to constrain color values to the range $[0, 1]$ [S1, unofficial][S2].

### Formulas
Let $x$ be the input 3D point.
Let $\gamma(x)$ be the Fourier positional encoding of $x$.
Let $f_\theta$ be the MLP function.

The density $\sigma$ and color $c$ are computed as:
$$ \sigma = \text{softplus}(W_\sigma h + b_\sigma) $$
$$ c = \text{sigmoid}(W_c h + b_c) $$

Where $h$ is the output of the MLP's hidden layers (with skip connections applied at `skip_layer`), and $W_\sigma, b_\sigma, W_c, b_c$ are the weights and biases of the respective linear heads [S1, unofficial][S2].

### Parameters
- `num_octaves`: 10 [S1, unofficial].
- `hidden_features`: 256 [S1, unofficial].
- `hidden_layers`: 4 [S1, unofficial].
- `skip_layer`: 2 [S1, unofficial].
- `iters`: 20000 training iterations [S1, unofficial].

### Edge Cases and Pitfalls
- **Non-negative Density**: Density must be non-negative for volume rendering. This is handled by using the softplus activation on the density head output [S1, unofficial][S2].
- **Color Range**: Color values must be in $[0, 1]$. This is handled by using the sigmoid activation on the color head output [S1, unofficial][S2].
- **View Dependence**: The model does not handle view dependence. If the scene has strong specular highlights or view-dependent materials, this model will produce blurred or inaccurate results [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class PositionOnlyNeRFField(nn.Module):
    def __init__(self, num_octaves=10, hidden_features=256, hidden_layers=4, skip_layer=2):
        super().__init__()
        self.num_octaves = num_octaves
        self.hidden_features = hidden_features
        self.hidden_layers = hidden_layers
        self.skip_layer = skip_layer
        
        # Calculate input dimension after positional encoding
        # Each of the 3 coordinates is encoded with 2*num_octaves frequencies (sin/cos)
        self.input_dim = 3 * 2 * num_octaves
        
        # Build hidden layers
        layers = []
        for i in range(hidden_layers):
            in_dim = self.input_dim if i == 0 else hidden_features
            out_dim = hidden_features
            layers.append(nn.Linear(in_dim, out_dim))
            if i < hidden_layers - 1: # ReLU for hidden layers, not the last one before heads
                layers.append(nn.ReLU())
        self.hidden_net = nn.Sequential(*layers)
        
        # Heads
        self.density_head = nn.Linear(hidden_features, 1)
        self.color_head = nn.Linear(hidden_features, 3)
        
    def forward(self, points_xyz):
        """
        points_xyz: [B, 3]
        returns: density [B, 1], color [B, 3]
        """
        # 1. Positional Encoding
        encoded = self._positional_encoding(points_xyz)
        
        # 2. MLP Forward with Skip Connections
        x = encoded
        for i, layer in enumerate(self.hidden_net.modules()):
            if isinstance(layer, nn.Linear):
                x = layer(x)
                # Apply ReLU if not the last linear layer of the hidden block
                # Note: The structure above puts ReLU after each linear except the last one
                # But we need to handle the skip connection at skip_layer
                if i == self.skip_layer:
                    # Concatenate original encoded position
                    x = torch.cat([x, encoded], dim=-1)
                # ReLU is applied by the Sequential module if it's not the last layer
                # However, the logic above is slightly flawed for strict layer-by-layer control
                # Let's rewrite the forward pass to be explicit about the skip connection logic
                pass
        
        # Rewritten explicit forward pass for clarity and correctness
        x = encoded
        for i in range(self.hidden_layers):
            linear_layer = self.hidden_net[i * 2] # Linear layer
            relu_layer = self.hidden_net[i * 2 + 1] if i < self.hidden_layers - 1 else None
            
            x = linear_layer(x)
            
            if i == self.skip_layer:
                x = torch.cat([x, encoded], dim=-1)
            
            if relu_layer is not None:
                x = relu_layer(x)
                
        # 3. Heads
        density = torch.softplus(self.density_head(x))
        color = torch.sigmoid(self.color_head(x))
        
        return density, color

    def _positional_encoding(self, x):
        """
        Standard Fourier feature mapping.
        x: [B, 3]
        returns: [B, 3 * 2 * num_octaves]
        """
        B, D = x.shape
        # Frequencies
        freq_bands = 2 ** torch.linspace(0, self.num_octaves - 1, self.num_octaves).to(x.device)
        # Scale factors for each octave
        # Typically, freq_bands are multiplied by 2^i for i in 0..num_octaves-1
        # And then multiplied by 2*pi
        # The input x is scaled by these frequencies
        
        # Reshape for broadcasting: [1, 1, num_octaves]
        freq_bands = freq_bands.view(1, 1, -1)
        
        # Expand x to [B, 3, num_octaves]
        x_expanded = x.unsqueeze(-1).expand(-1, -1, self.num_octaves)
        
        # Multiply by frequency bands
        x_scaled = x_expanded * freq_bands * 2 * math.pi
        
        # Flatten the last two dimensions
        x_scaled = x_scaled.reshape(B, -1)
        
        # Apply sin and cos
        encoded = torch.cat([torch.sin(x_scaled), torch.cos(x_scaled)], dim=-1)
        return encoded
```

## Subtopics

### [Unbounded inward-facing scene parameterization](957-unbounded-inward-facing-scene-parameterization.md)
Parameterizes the scene to handle unbounded volumes, unlike the bounded assumption often implicit in simple MLPs.

### [Tiny NeRF](1275-tiny-nerf.md)
A further compressed version of the NeRF MLP, reducing parameters even more than the position-only variant.

### [MLPRender_Fea](2414-mlprender-fea.md)
Uses feature vectors from a 3D grid to condition the MLP, allowing for view-dependent effects and better detail.

### [View-Dependent MLP](2444-view-dependent-mlp.md)
The full NeRF architecture that includes viewing direction as input to model specularities and view-dependent shading.

## Operating characteristics
- **Time Complexity**: Training is faster than view-dependent NeRFs due to the smaller model size and lack of direction input processing [S1, unofficial].
- **Memory Complexity**: Lower memory footprint due to fewer parameters [S1, unofficial].
- **Performance**: Results may be blurred for scenes with complex lighting or view-dependent effects [S1, unofficial].
- **Strengths**: Fast training, low memory usage, simple implementation [S1, unofficial].
- **Limitations**: Cannot model view-dependent appearance [S1, unofficial].
- **When to use**: Scenes with uniform lighting or when computational resources are limited [S1, unofficial].
- **When not to use**: Scenes with strong specular highlights or view-dependent materials [S1, unofficial].

## Background
The position-only NeRF field is a simplified variant of the Neural Radiance Field (NeRF) introduced by Mildenhall et al. in 2020 [S1, unofficial]. It builds on the concept of implicit neural representations, specifically coordinate MLPs with skip connections, as explored by Sitzmann et al. in 2020 (SIREN) [S1, unofficial]. The position-only variant was used in early implementations to demonstrate the core volume rendering pipeline with reduced complexity [S1, unofficial].

## Sources
[S1] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S2] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
