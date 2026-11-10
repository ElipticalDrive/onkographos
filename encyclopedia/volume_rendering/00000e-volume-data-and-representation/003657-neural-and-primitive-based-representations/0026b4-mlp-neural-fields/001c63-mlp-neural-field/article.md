# MLP Neural Field

Also called: Baseline MLP, Coordinate MLP, Plain MLP, Plain MLP Neural Field, ReLU-MLP

## Overview
A plain MLP neural field is a coordinate-based neural network that maps continuous spatial coordinates to signal values (such as density and color) using standard feedforward layers and smooth activations like ReLU [S1, unofficial]. It serves as the primary baseline for evaluating neural fields, particularly against architectures designed to overcome spectral bias, such as SIREN [S1, unofficial]. Unlike its siblings, it relies on standard initialization and activation functions without specialized frequency handling or input skip connections [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A batch of continuous coordinate vectors $\mathbf{x} \in \mathbb{R}^d$ (typically $d=3$ for spatial position). The coordinates follow the $[-1, 1]$ convention [S1, unofficial].
*   **Output:** A signal vector $\mathbf{y} \in \mathbb{R}^k$ (e.g., density $\sigma$ and color $\mathbf{c}$) [S1, unofficial].

**Data Structures:**
The network is a sequential stack of linear layers and activations [S1, unofficial]:
1.  First layer: $Linear(in\_features, hidden\_features)$
2.  Hidden layers: $(hidden\_layers - 1)$ repetitions of $Linear(hidden\_features, hidden\_features)$ followed by an activation.
3.  Output layer: $Linear(hidden\_features, out\_features)$

**Parameters:**
*   `in_features`: Dimensionality of input coordinates [S1, unofficial].
*   `out_features`: Dimensionality of output values [S1, unofficial].
*   `hidden_features`: Width of hidden layers [S1, unofficial].
*   `hidden_layers`: Number of hidden layers [S1, unofficial].
*   `bias`: Boolean, default `True`. Whether linear layers carry a bias [S1, unofficial].
*   `activation`: String, default `"ReLU"`. Name of the activation function (e.g., ReLU, Tanh, GELU) [S1, unofficial].
*   `steps`: Integer, default `2000`. Number of training steps for fitting an image with ReLU-MLP [S1, unofficial].

**Algorithm:**
1.  Clone the input coordinate tensor $\mathbf{x}$ to create $\mathbf{x}_{grad}$.
2.  Enable gradient tracking on $\mathbf{x}_{grad}$ to allow differentiation with respect to coordinates [S1, unofficial].
3.  Pass $\mathbf{x}_{grad}$ through the MLP architecture defined above.
4.  Return the output signal and $\mathbf{x}_{grad}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class MLPNeuralField(nn.Module):
    def __init__(self, in_features, out_features, hidden_features, hidden_layers, bias=True, activation="ReLU"):
        super().__init__()
        layers = []
        # First layer
        layers.append(nn.Linear(in_features, hidden_features, bias=bias))
        layers.append(get_activation(activation))
        
        # Hidden layers
        for _ in range(hidden_layers - 1):
            layers.append(nn.Linear(hidden_features, hidden_features, bias=bias))
            layers.append(get_activation(activation))
            
        # Output layer
        layers.append(nn.Linear(hidden_features, out_features, bias=bias))
        
        self.network = nn.Sequential(*layers)

    def forward(self, x):
        # Clone input and enable gradients for coordinate differentiation
        x_grad = x.clone().detach().requires_grad_(True)
        output = self.network(x_grad)
        return output, x_grad

def get_activation(name):
    if name == "ReLU":
        return nn.ReLU()
    elif name == "Tanh":
        return nn.Tanh()
    elif name == "GELU":
        return nn.GELU()
    else:
        raise ValueError(f"Activation {name} not supported")
```

**Edge Cases and Pitfalls:**
*   **Spectral Bias:** Standard ReLU-MLPs struggle to learn high-frequency details compared to sine-activated networks (SIREN), often failing to reach sharp reconstructions within standard training steps (e.g., 2000 steps) [S1, unofficial].
*   **Coordinate Convention:** Inputs must be normalized to the $[-1, 1]$ range to ensure proper gradient flow and activation behavior [S1, unofficial].
*   **Gradient Requirement:** The input tensor must be cloned and have gradients enabled if the rendering or loss function requires derivatives with respect to spatial coordinates (e.g., for volume rendering integrals or sphere tracing) [S1, unofficial].

## Subtopics

### [MLP with Input Skips](1643-mlp-with-input-skips.md)
An MLP architecture that concatenates the original input $\mathbf{x}$ to intermediate layers specified in `input_skips`. It allows the network to access low-level features at deeper layers. Choose this when the baseline MLP fails to capture high-frequency details due to spectral bias, as it mitigates information loss in deep networks. Link: [MLP with Input Skips](1643-mlp-with-input-skips.md).

### [Residual Block with Shortcut (ReLU MLP Extension)](10016-residual-block-with-shortcut-relu-mlp-extension.md)
This method extends the analytic marching algorithm for implicit surface reconstruction to MLPs constructed with residual blocks containing shortcut connections. It enables theoretical analysis and exact linear region partitioning, allowing for efficient meshing via analytic marching. It differs from the baseline by explicitly handling the additive shortcut path in the region-wise linear mapping, and from [MLP with Input Skips](1643-mlp-with-input-skips.md) by using a residual summation structure rather than concatenation or direct input reuse.

### MLP Decomposition Baseline
An alternative decomposition method where a neural network (MLP) with a softmax activation outputs the weights $w_{\phi n}(\mathbf{x})$. It is used as a baseline to demonstrate the efficiency of Voronoi decomposition [S2]. While it has similar theoretical operation counts to Voronoi-based methods, it suffers from random memory access during integration, leading to much faster runtime growth as the number of heads increases [S2].

### Deformation MLP Configuration
A specific configuration of the MLP used for deformation in dynamic radiance fields. It consists of 8 layers with 256 neurons each, using positional encoding frequencies of $L_x = 10$ for spatial coordinates and $L_t = 10$ for time [S3]. The input is time $t$ and Gaussian 3D position $\mathbf{x}_k$, and the output is delta position $\delta\mathbf{x}_k$ [S3]. Biases are initialized to zero, weights with Xavier normal initialization, and the final layer weights with a normal distribution ($\sigma=10^{-5}$) to ensure stable optimization by starting deformations from small values [S3].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of hidden layers and width. Training time is significantly longer than SIREN for high-frequency scenes; e.g., 2000 steps may not achieve what SIREN does in ~100 steps [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of parameters: $(in\_features \times hidden\_features) + (hidden\_layers \times hidden\_features^2) + (hidden\_features \times out\_features)$ [S1, unofficial].
*   **Strengths:** Simple to implement; serves as a robust baseline for comparison; compatible with standard deep learning frameworks and optimizers [S1, unofficial].
*   **Limitations:** Suffers from spectral bias, making it unsuitable for high-frequency detail reconstruction without architectural modifications (like input skips or SIREN activations) [S1, unofficial].
*   **When to use:** As a baseline for comparison in neural field research; for scenes with low-frequency content; when computational simplicity is prioritized over reconstruction quality.
*   **When not to use:** For high-frequency detail reconstruction; when training time is constrained and SIREN-like convergence is needed.

## Background
The plain MLP neural field was introduced as a baseline method in the context of differentiable rendering and neural implicit representations [S1, unofficial]. It is used to contrast with specialized architectures like SIREN, which use sine activations to handle high-frequency signals [S1, unofficial]. The method is foundational in the study of spectral bias in coordinate-based neural networks [S1, unofficial].

## Sources
[S1] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S2] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26
[S3] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
