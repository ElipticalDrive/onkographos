# MLP Neural Field

Also called: Baseline MLP, Coordinate MLP, Plain MLP, Plain MLP Neural Field, ReLU-MLP

## Overview
A plain MLP neural field maps continuous coordinates to signal values using a standard Multi-Layer Perceptron with a configurable activation, typically ReLU [S1, unofficial]. It serves as the baseline against SIREN [1250-siren-neural-field.md], contrasting with architectures that use sinusoidal activations to represent high-frequency detail and have well-behaved derivatives [S1, unofficial]. Unlike [MLP with Input Skips](1643-mlp-with-input-skips.md), the plain variant does not concatenate the original input vector to intermediate layers [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tensor of continuous coordinates $\mathbf{x} \in \mathbb{R}^{N \times D_{in}}$, where $N$ is the batch size and $D_{in}$ is the spatial dimensionality (`in_features`) [S1, unofficial].
*   **Output:** A tensor of signal values $\mathbf{y} \in \mathbb{R}^{N \times D_{out}}$, where $D_{out}$ is the output dimensionality (`out_features`) [S1, unofficial].

**Data Structures:**
The network is a sequential stack of linear layers and activation functions with the exact layout: `[Linear(in_features, hidden_features), Activation, (Linear(hidden_features, hidden_features), Activation) \times (hidden\_layers - 1), Linear(hidden_features, out_features)]` [S1, unofficial].

**Numbered Steps:**
1. Clone the input coordinate tensor $\mathbf{x}$ to create $\mathbf{x}'$ and enable gradient tracking on $\mathbf{x}'$ [S1, unofficial].
2. Pass $\mathbf{x}'$ through the sequential network layers [S1, unofficial].
3. Return the output tensor $\mathbf{y}$ and the gradient-enabled coordinate tensor $\mathbf{x}'$ [S1, unofficial].

**Formulas:**
For each hidden layer $i$ ($1 \le i \le hidden\_layers$):
$$h_i = \text{ReLU}(W_i h_{i-1} + b_i)$$
where $h_0 = \mathbf{x}'$, $W_i$ and $b_i$ are the weight matrix and bias vector for layer $i$, and $\text{ReLU}(z) = \max(0, z)$ [G].
The final output is computed as:
$$\mathbf{y} = W_{out} h_{hidden\_layers} + b_{out}$$
where $W_{out}$ and $b_{out}$ are the output layer weights and bias [G].

**Parameters:**
*   `in_features` ($D_{in}$): Dimensionality of input coordinates [S1, unofficial].
*   `out_features` ($D_{out}$): Dimensionality of output values [S1, unofficial].
*   `hidden_features` ($H$): Width of hidden layers [S1, unofficial].
*   `hidden_layers` ($L$): Number of hidden layers [S1, unofficial].
*   `bias` ($bool$): Whether linear layers include a bias term (default: True) [S1, unofficial].
*   `activation` ($str$): Name of the activation function (default: "ReLU") [S1, unofficial].
*   `steps` ($int$): Number of training steps for fitting an image (default: 2000) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class PlainMLP(nn.Module):
    def __init__(self, in_features, out_features, hidden_features, hidden_layers, bias=True, activation="ReLU"):
        super().__init__()
        layers = []
        layers.append(nn.Linear(in_features, hidden_features, bias=bias))
        layers.append(_get_activation(activation))
        for _ in range(hidden_layers - 1):
            layers.append(nn.Linear(hidden_features, hidden_features, bias=bias))
            layers.append(_get_activation(activation))
        layers.append(nn.Linear(hidden_features, out_features, bias=bias))
        self.network = nn.Sequential(*layers)

    def forward(self, x):
        x_grad = x.clone().detach().requires_grad_(True)
        out = self.network(x_grad)
        return out, x_grad

def _get_activation(name):
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
*   **Coordinate Convention:** Inputs must follow the $[-1, 1]$ normalization convention [S1, unofficial].
*   **Gradient Tracking:** The input tensor must be cloned and have gradients enabled; otherwise, computing spatial derivatives will fail [S1, unofficial].
*   **Spectral Bias:** The plain ReLU-MLP suffers from spectral bias, struggling to learn high-frequency signals compared to SIREN [S1, unofficial].

## Subtopics

### [MLP with Input Skips](1643-mlp-with-input-skips.md)
An MLP architecture that concatenates the original input vector to intermediate layers, allowing the network to access low-level features at deeper layers. It serves as the baseline for standard NeRF-style neural fields, contrasting with SIREN by using ReLU activations and explicit skip connections rather than sinusoidal embeddings.

### [Deformation MLP Configuration](9661-deformation-mlp-configuration.md)
The Deformation MLP Configuration defines the architecture, initialization, and conditioning mechanisms for a Multi-Layer Perceptron (MLP) used to predict spatial or temporal deformations in neural volume representations. It solves the problem of mapping canonical coordinates to deformed coordinates by providing a structured way to inject instance-specific or time-specific information into the network's feature space via feature-wise linear modulation. It is distinguished from sibling methods like [MLP with Input Skips](1643-mlp-with-input-skips.md) by its explicit use of feature-wise modulation to handle conditional inputs, and from [Residual Block with Shortcut (ReLU MLP Extension)](10016-residual-block-with-shortcut-relu-mlp-extension.md) by its focus on deformation dynamics rather than static residual learning.

### [Residual Block with Shortcut (ReLU MLP Extension)](10016-residual-block-with-shortcut-relu-mlp-extension.md)
This method extends the analytic marching algorithm for implicit surface reconstruction to Multi-Layer Perceptrons (MLPs) constructed with residual blocks containing shortcut connections. It enables the theoretical analysis and exact linear region partitioning of such architectures, allowing for efficient meshing via analytic marching.

### MLP Decomposition Baseline
An alternative decomposition method where a neural network (MLP) with a softmax activation outputs the weights $w_{\phi n}(\mathbf{x})$ [S2]. It is used as a baseline to demonstrate the efficiency of Voronoi decomposition [S2]. While it has similar theoretical operation counts to Voronoi decomposition, it suffers from random memory access during integration, leading to much faster runtime growth as the number of heads increases [S2].

## Operating characteristics
*   **Training Time:** Fitting an image requires ~2000 steps for the ReLU-MLP, significantly more than the ~100 steps required by SIREN [S1, unofficial].
*   **Strengths:** Simple to implement; standard ReLU activations are computationally efficient and widely supported [S1, unofficial].
*   **Limitations:** Poor at representing high-frequency details due to spectral bias [S1, unofficial].
*   **When to Use:** As a baseline for comparison; for low-frequency or smooth scenes [S1, unofficial].
*   **When Not to Use:** For high-frequency details (e.g., sharp edges, fine textures) [S1, unofficial].

## Background
The plain MLP neural field was introduced as a baseline method in early neural rendering research to contrast with the SIREN (Sinusoidal Representation Networks) architecture [S1, unofficial]. While SIREN demonstrated the ability to represent high-frequency signals using sine activations, the ReLU-MLP highlighted the limitations of standard feedforward networks in this domain, specifically spectral bias [S1, unofficial].

## Sources
[S1] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S2] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26
