# SIREN Neural Field
Also called: 3D Implicit Neural Representation (INR) Module, SIREN, Sine Neural Field, Sinusoidal Representation Network

## Overview
The SIREN (Sinusoidal Representation Network) is a neural field that maps continuous coordinates to signal values using sine activations with a specific initialization, allowing it to represent high-frequency detail and have well-behaved derivatives [S2, unofficial]. It solves the problem of spectral bias in standard Multi-Layer Perceptrons (MLPs), which struggle to learn high-frequency signals [S2, unofficial]. Unlike the [MLP Neural Field](1256-mlp-neural-field.md) which typically uses ReLU or similar smooth activations, SIREN uses sine activations throughout (or in the final layer) to enable sharp reconstruction, reaching convergence in ~100 steps where ReLU-MLPs fail even after 2000 steps [S2, unofficial].

## How it works
**Inputs and Outputs:**
* Input: Continuous coordinates $\mathbf{x} \in \mathbb{R}^{d_{in}}$ (typically 2D for images, 3D for volumes) [S2, unofficial].
* Output: Signal values $\mathbf{y} \in \mathbb{R}^{d_{out}}$ (e.g., intensity, density, color) [S2, unofficial].

**Data Structures:**
The network is a sequential stack of layers [S2, unofficial]. Each layer is defined by its input dimension, output dimension, and whether it is the first layer [S2, unofficial].
* `SineLayer`: A linear transformation followed by a sine activation [S2, unofficial].
    * Fields: `in_features` (int), `out_features` (int), `is_first` (bool), `omega` (float) [S2, unofficial].
* `Linear`: A standard linear transformation (affine if bias is True) [S2, unofficial].
    * Fields: `in_features` (int), `out_features` (int), `bias` (bool) [S2, unofficial].

**Parameters:**
* `in_features`: Dimensionality of input coordinates [S2, unofficial].
* `out_features`: Dimensionality of output values [S2, unofficial].
* `hidden_features`: Width of hidden layers [S2, unofficial].
* `hidden_layers`: Number of hidden sine layers [S2, unofficial].
* `bias`: Whether linear layers carry a bias (default: `True`) [S2, unofficial].
* `last_layer_linear`: If `True`, the final layer is linear instead of sine (default: `False`) [S2, unofficial].
* `first_omega_0`: Frequency parameter for the first layer (default: 20.0) [S2, unofficial].
* `hidden_omega_0`: Frequency parameter for hidden layers (default: 20.0) [S2, unofficial].

**Initialization:**
The weights are initialized uniformly to ensure the sine activations operate in a regime where their derivative is non-zero and well-behaved [S2, unofficial].
1. **First Layer:** Weights $W_1$ are sampled from $U(-\frac{1}{n}, \frac{1}{n})$, where $n = \text{in\_features}$ [S2, unofficial].
2. **Hidden Layers:** For layer $i > 1$, weights $W_i$ are sampled from $U(-\frac{\sqrt{6/n}}{\omega_0}, \frac{\sqrt{6/n}}{\omega_0})$, where $n = \text{in\_features}$ of that layer and $\omega_0$ is the appropriate frequency parameter (`first_omega_0` for the first hidden layer, `hidden_omega_0` for subsequent ones) [S2, unofficial].
3. **Final Layer (if linear):** If `last_layer_linear` is `True`, weights $W_{last}$ are sampled from $U(-\frac{\sqrt{6/n}}{\omega_{last}}, \frac{\sqrt{6/n}}{\omega_{last}})$, where $n = \text{hidden\_features}$ and $\omega_{last} = \text{hidden\_omega\_0}$ [S2, unofficial].
4. **Final Layer (if sine):** If `last_layer_linear` is `False`, the final layer is a `SineLayer` initialized like a hidden layer [S2, unofficial].

**Forward Pass:**
The input coordinates are cloned and gradients are enabled on the copy to allow differentiation with respect to the input coordinates (e.g., for computing gradients or Laplacians) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
import torch.nn as nn

class SineLayer(nn.Module):
    """A linear layer followed by a sine activation."""
    def __init__(self, in_features, out_features, is_first=False, omega=20.0):
        super().__init__()
        self.omega = omega
        self.is_first = is_first
        self.linear = nn.Linear(in_features, out_features, bias=True)
        self.init_weights()

    def init_weights(self):
        with torch.no_grad():
            if self.is_first:
                bound = 1.0 / self.linear.in_features
                self.linear.weight.uniform_(-bound, bound)
            else:
                bound = math.sqrt(6.0 / self.linear.in_features) / self.omega
                self.linear.weight.uniform_(-bound, bound)

    def forward(self, x):
        return torch.sin(self.linear(x) * self.omega)

class SIREN(nn.Module):
    """SIREN Neural Field."""
    def __init__(self, in_features, out_features, hidden_features, hidden_layers,
                 bias=True, last_layer_linear=False, first_omega_0=20.0, hidden_omega_0=20.0):
        super().__init__()
        layers = []
        layers.append(SineLayer(in_features, hidden_features, is_first=True, omega=first_omega_0))
        for _ in range(hidden_layers):
            layers.append(SineLayer(hidden_features, hidden_features, is_first=False, omega=hidden_omega_0))
        if last_layer_linear:
            final_linear = nn.Linear(hidden_features, out_features, bias=bias)
            bound = math.sqrt(6.0 / hidden_features) / hidden_omega_0
            with torch.no_grad():
                final_linear.weight.uniform_(-bound, bound)
            layers.append(final_linear)
        else:
            layers.append(SineLayer(hidden_features, out_features, is_first=False, omega=hidden_omega_0))
        self.network = nn.Sequential(*layers)

    def forward(self, x):
        x_clone = x.clone().detach().requires_grad_(True)
        output = self.network(x_clone)
        return output, x_clone
```

**Edge Cases and Pitfalls:**
* **Coordinate Convention:** Inputs should follow the $[-1, 1]$ convention [S2, unofficial].
* **Gradient Computation:** The input must be cloned with gradients enabled if derivatives with respect to coordinates are required [S2, unofficial].
* **Spectral Bias:** Without sine activations and specific initialization, the network suffers from spectral bias, failing to learn high-frequency details [S2, unofficial].

## Subtopics

### [Fourier Positional Encoding](1267-fourier-positional-encoding.md)
Maps low-dimensional input coordinates into a higher-dimensional space using fixed sinusoidal bases. It sets itself apart from SIREN by providing explicit frequency bands as input features rather than relying on the network's internal sine layers. Choose it when you need to lift raw coordinates into a Fourier space before passing them to an MLP.

## Operating characteristics
* **Time Complexity:** Training time is significantly faster than ReLU-MLPs for high-frequency signals. SIREN reaches sharp reconstruction in ~100 steps, whereas ReLU-MLPs do not converge in 2000 steps [S2, unofficial].
* **Memory Complexity:** The method stores network parameters instead of explicit voxels, substantially cutting GPU memory usage compared to conventional explicit representations [S1].
* **Strengths:** Well-behaved derivatives, ability to represent high-frequency details [S2, unofficial].
* **Limitations:** Sensitive to initialization [S2, unofficial]. Not given in the sources for specific hardware performance metrics.
* **When to use:** Not given in the sources.
* **When not to use:** Not given in the sources.

## Background
The SIREN is a neural field method that addresses the limitations of standard MLPs in representing continuous signals, particularly in the context of neural fields for volume rendering and image representation [S2, unofficial]. The method leverages the properties of sine functions to overcome spectral bias, a common issue in deep learning where networks tend to learn low-frequency functions first [S2, unofficial]. It is employed in architectures for encoding volumetric data and super-resolution tasks [S1]. The specific origin paper, year, and detailed historical development are not given in the sources.

## Sources
[S1] An end-to-end implicit neural representation architecture for medical volume data., https://doi.org/10.1371/journal.pone.0314944, fetched 2026-10-26
[S2] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
