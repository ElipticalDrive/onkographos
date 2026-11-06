# MLP with Input Skips

Also called: Skip Connection MLP

## Overview
An MLP architecture that concatenates the original input $z$ to intermediate layers specified in `input_skips`, similar to the skip connections in original NeRF [S1, unofficial]. It allows the network to access low-level features at deeper layers [S2, unofficial]. It builds on [MLP Neural Field](1256-mlp-neural-field.md) by explicitly injecting the raw coordinate information into hidden representations, contrasting with the plain MLP which only passes transformed features forward.

## How it works
**Inputs and Outputs:**
- Input: A vector $z \in \mathbb{R}^{skip\_dim}$ (typically 3D spatial coordinates).
- Output: A vector $y_{out} \in \mathbb{R}^{output\_dim}$.

**Data Structures:**
- `input_skips`: A list of integers representing layer indices where the input $z$ is concatenated.
- `n_layers`: Total number of linear layers.
- `hidden_dim`: Dimensionality of the hidden layers.
- `skip_dim`: Dimensionality of the input vector $z$.
- `output_dim`: Dimensionality of the output vector.

**Parameters:**
- `input_dim`: Dimension of the input vector. Note: In this architecture, `input_dim` is effectively `skip_dim` for the first layer, but subsequent layers expect `hidden_dim` or `hidden_dim + skip_dim`.
- `n_layers`: Total number of linear layers in the MLP [S1, unofficial][S2, unofficial].
- `output_dim`: Dimension of the output vector [S1, unofficial].
- `skip_dim`: Dimension of the skip connection input $z$ [S1, unofficial][S2, unofficial].
- `hidden_dim`: Dimensionality of the hidden layers [S1, unofficial][S2, unofficial].

**Steps:**
1. Initialize the hidden state $y$ with the input $z$.
2. Iterate through each layer $i$ from $0$ to $n\_layers - 1$:
   a. If $i$ is in `input_skips`, concatenate $z$ to $y$: $y = [y, z]$.
   b. Apply linear transformation: $y = W_i y + b_i$.
   c. Apply ReLU activation: $y = \text{ReLU}(y)$.
3. After the final layer, apply a final linear transformation (if `output_dim` differs from `hidden_dim`) or return $y$ directly if the last layer outputs `output_dim`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class MLPWithInputSkips(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim, n_layers, input_skips):
        super().__init__()
        self.input_dim = input_dim
        self.hidden_dim = hidden_dim
        self.output_dim = output_dim
        self.n_layers = n_layers
        self.input_skips = input_skips
        
        # Build layers
        self.layers = nn.ModuleList()
        
        # Layer 0
        # Input is input_dim (which is skip_dim)
        self.layers.append(nn.Linear(input_dim, hidden_dim))
        
        # Intermediate layers
        for i in range(1, n_layers):
            if i in input_skips:
                # Input is hidden_dim + skip_dim
                self.layers.append(nn.Linear(hidden_dim + input_dim, hidden_dim))
            else:
                # Input is hidden_dim
                self.layers.append(nn.Linear(hidden_dim, hidden_dim))
                
        # Final layer
        # The last layer should output output_dim
        # If the last layer index is in input_skips, it takes hidden_dim + skip_dim
        # Otherwise, it takes hidden_dim
        last_idx = n_layers - 1
        if last_idx in input_skips:
            self.layers.append(nn.Linear(hidden_dim + input_dim, output_dim))
        else:
            self.layers.append(nn.Linear(hidden_dim, output_dim))

    def forward(self, z):
        y = z
        for i, layer in enumerate(self.layers):
            if i in self.input_skips:
                # Concatenate original input z to current feature map y
                y = torch.cat([y, z], dim=-1)
            y = layer(y)
            # Apply ReLU activation for all layers except possibly the last one?
            # Sources say "Linear layer followed by ReLU activation" for constructing layers.
            # Usually the final layer does not have ReLU for regression tasks like color/density.
            # However, S1 says "Construct each layer as a Linear layer followed by ReLU activation."
            # S2 says "apply linear transformation and ReLU".
            # We will apply ReLU to all layers as per S1/S2 general description, 
            # but note that in practice, the final layer often omits ReLU.
            # Given the strict instruction to follow sources, we apply ReLU.
            # [G] In standard NeRF implementations, the final layer typically does not use ReLU.
            # If the source implies all layers, we follow that. 
            # Let's assume ReLU is applied to all layers as per S1/S2 text.
            y = torch.relu(y)
        return y
```

**Edge Cases and Pitfalls:**
- `input_skips` must contain valid layer indices $< n\_layers$.
- Precision issues may arise if `hidden_dim` is very large, causing gradient vanishing/exploding without proper initialization.
- Boundary conditions: Ensure `z` is broadcastable or concatenated correctly along the feature dimension.

## Subtopics

### [ODENetwork (Conditioned Neural ODE)](5241-odenetwork-conditioned-neural-ode.md)
ODENetwork is a neural field architecture that computes the derivative of a latent state over a scalar time parameter, conditioned on a voxel embedding. It serves as the core differentiable function for continuous normalizing flows in voxel-conditioned generation, transforming a base latent distribution into a target distribution by integrating the ODE from $t=0$ to $t=1$. It builds on [MLP with Input Skips](1643-mlp-with-input-skips.md) by explicitly incorporating time and voxel conditions into the skip connections.

## Operating characteristics
- Time complexity: $O(n\_layers \cdot hidden\_dim^2)$ per forward pass, assuming `input_skips` is sparse. If dense, $O(n\_layers \cdot (hidden\_dim + skip\_dim) \cdot hidden\_dim)$.
- Memory complexity: $O(n\_layers \cdot hidden\_dim^2)$ for weights.
- Strengths: Improves gradient flow and allows the network to learn high-frequency details by providing direct access to input coordinates.
- Limitations: Increases parameter count slightly due to larger input dimensions for skipped layers.
- When to use: When the target function has high-frequency components or when plain MLPs struggle to converge.
- When not to use: When computational resources are extremely limited and the function is smooth.

## Background
Originated from the original NeRF paper [S1, unofficial]. It is a variant of the MLP Neural Field that introduces skip connections to improve training stability and reconstruction quality [S2, unofficial].

## Sources
[S1] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S2] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
