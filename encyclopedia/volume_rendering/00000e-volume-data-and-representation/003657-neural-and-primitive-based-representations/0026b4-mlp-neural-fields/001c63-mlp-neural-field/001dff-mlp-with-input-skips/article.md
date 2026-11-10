# MLP with Input Skips

Also called: Skip Connection MLP

## Overview
An MLP architecture that concatenates the original input $z$ to intermediate layers specified in `input_skips` [S1, unofficial][S2, unofficial]. It allows the network to access low-level features at deeper layers [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A vector $z \in \mathbb{R}^{skip\_dim}$ (typically 3D spatial coordinates) [S1, unofficial][S2, unofficial].
- Output: A vector $y_{out} \in \mathbb{R}^{output\_dim}$ [S1, unofficial][S2, unofficial].

**Data Structures:**
- `input_skips`: A list of integers representing layer indices where the input $z$ is concatenated [S2, unofficial].
- `n_layers`: Total number of linear layers [S1, unofficial][S2, unofficial].
- `hidden_dim`: Dimensionality of the hidden layers [S1, unofficial][S2, unofficial].
- `skip_dim`: Dimensionality of the input vector $z$ [S1, unofficial][S2, unofficial].
- `output_dim`: Dimensionality of the output vector [S1, unofficial][S2, unofficial].

**Parameters:**
- `input_dim`: Dimension of the input vector, equivalent to `skip_dim` for the first layer [S2, unofficial].
- `n_layers`: Total number of linear layers in the MLP [S1, unofficial][S2, unofficial].
- `output_dim`: Dimension of the output vector [S1, unofficial].
- `skip_dim`: Dimension of the skip connection input $z$ [S1, unofficial][S2, unofficial].
- `hidden_dim`: Dimensionality of the hidden layers [S1, unofficial][S2, unofficial].

**Steps:**
1. Initialize the hidden state $y$ with the input $z$ [S2, unofficial].
2. Iterate through each layer index $i$ from $0$ to $n\_layers - 1$:
   a. If $i$ is in `input_skips`, concatenate $z$ to $y$: $y = [y, z]$ [S2, unofficial].
   b. Apply linear transformation and ReLU activation: $y = \text{ReLU}(W_i y + b_i)$ [S2, unofficial].
3. Return the final hidden state $y$ [S1, unofficial].

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
        self.input_skips = set(input_skips)
        self.layers = nn.ModuleList()

        for i in range(n_layers):
            is_last = (i == n_layers - 1)
            out_dim = output_dim if is_last else hidden_dim
            if i == 0:
                in_dim = input_dim
            elif i in self.input_skips:
                in_dim = hidden_dim + input_dim
            else:
                in_dim = hidden_dim
            self.layers.append(nn.Linear(in_dim, out_dim))

    def forward(self, z):
        y = z
        for i, layer in enumerate(self.layers):
            if i in self.input_skips:
                y = torch.cat([y, z], dim=-1)
            y = layer(y)
            y = torch.relu(y)
        return y
```

**Edge Cases and Pitfalls:**
- `input_skips` must contain valid layer indices strictly less than `n_layers` [S2, unofficial].
- Precision issues may arise if `hidden_dim` is very large, causing gradient vanishing or exploding without proper initialization [G].
- Boundary conditions: Ensure `z` is broadcastable or concatenated correctly along the feature dimension [G].

## Subtopics

### [ODENetwork (Conditioned Neural ODE)](5241-odenetwork-conditioned-neural-ode.md)
ODENetwork is a neural field architecture that computes the derivative of a latent state over a scalar time parameter, conditioned on a voxel embedding. It serves as the core differentiable function for continuous normalizing flows in voxel-conditioned generation, transforming a base latent distribution into a target distribution by integrating the ODE from $t=0$ to $t=1$. It builds on [MLP with Input Skips](1643-mlp-with-input-skips.md) by explicitly incorporating time and voxel conditions into the skip connections.

## Operating characteristics
- Time complexity: Not given in the sources.
- Memory complexity: Not given in the sources.
- Strengths: Not given in the sources.
- Limitations: Not given in the sources.
- When to use: Not given in the sources.
- When not to use: Not given in the sources.

## Background
A neural field architecture variant that introduces input skip connections [S2, unofficial].

## Sources
[S1] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S2] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
