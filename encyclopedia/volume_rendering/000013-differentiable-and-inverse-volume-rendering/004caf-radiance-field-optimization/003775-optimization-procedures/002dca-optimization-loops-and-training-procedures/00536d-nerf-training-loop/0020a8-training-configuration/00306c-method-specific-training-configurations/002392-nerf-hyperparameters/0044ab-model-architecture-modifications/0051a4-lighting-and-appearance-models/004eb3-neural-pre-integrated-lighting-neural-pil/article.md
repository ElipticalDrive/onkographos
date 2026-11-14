# Neural Pre-Integrated Lighting (Neural-PIL)
Also called: Neural-PIL

## Overview
Neural-PIL is a neural network module that replaces the costly numerical integration of incoming light with a simple network query, predicting pre-integrated light $\tilde{L}_i(\omega_r, b_r)$ directly from an illumination embedding, direction, and roughness [S1, unofficial]. Unlike spherical Gaussian (SG) or spherical harmonic (SH) light representations which suffer from lack of fine details or high memory costs, Neural-PIL is an MLP that directly produces pre-integrated light, representing finer details in illumination [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Illumination embedding $\mathbf{z}_l \in \mathbb{R}^{128}$ [S1, unofficial].
    *   Incoming light direction (mirrored view direction) $\omega_r \in \mathbb{R}^3$ [S1, unofficial].
    *   Roughness $b_r$ (scalar) [S1, unofficial].
*   **Output:**
    *   Pre-integrated light $\tilde{L}_i(\omega_r, b_r)$ [S1, unofficial].

**Data Structures**
The architecture consists of:
1.  **Neural-PIL Network:** A coordinate-based MLP using FiLM-SIREN layers.
2.  **Mapping Network 1:** Predicts modulating parameters for the first few layers of the Neural-PIL based on $\mathbf{z}_l$.
3.  **Mapping Network 2:** Predicts modulating parameters for the penultimate layer of the Neural-PIL based on $b_r$.
4.  **Light-SMAE (Light Smooth Manifold Auto-encoder):** A CNN encoder/decoder used to generate $\mathbf{z}_l$ from environment maps. This is trained separately and frozen during Neural-PIL training to ease memory requirements [S1, unofficial].

**Formulas**
The pre-integrated light $\tilde{L}_i(\omega_r, b_r)$ is defined as:
$$
\tilde{L}_i(\omega_r, b_r) = \int_{\Omega} D(b_r, \omega_i, \omega_r) L_i(x, \omega_i) d\omega_i
$$
where $D$ describes the microfacet distribution [S1, unofficial].

The rendered radiance $L_o(x, \omega_o)$ approximates the full BRDF integration using this pre-integrated light:
$$
L_o(x, \omega_o) \approx (b_d / \pi)\tilde{L}_i(n, 1) + b_s (F_0(\omega_o, n)B_0(\omega_o \cdot n, b_r) + B_1(\omega_o \cdot n, b_r))\tilde{L}_i(\omega_r, b_r)
$$
where:
*   $b_d, b_s$ are diffuse and specular albedo components.
*   $F_0(\omega_o, n) = (1 - \omega_o \cdot h)^5$ with $h = (\omega_i + \omega_o)/\|\omega_i + \omega_o\|$ [S1, unofficial].
*   $B_0, B_1$ are 2D lookup textures indexed by $(\omega_o \cdot n)$ and roughness $b_r$ [S1, unofficial].
*   The diffuse part queries $\tilde{L}_i(n, 1)$ (roughness 1), while the specular part queries $\tilde{L}_i(\omega_r, b_r)$ [S1, unofficial].

The core activation function for the Neural-PIL layers is the FiLM-SIREN layer:
$$
\phi(y) = \sin(\lambda_0 \gamma \odot y + \beta)
$$
where $\odot$ denotes the Hadamard product, and $\lambda_0, \beta, \gamma$ are modulating parameters [S1, unofficial].

**Steps**
1.  **Encode Illumination:** Map a 2D environment map (e.g., $128 \times 256$) to a 128-dimensional smooth latent space $\mathbf{z}_l$ using the frozen Light-SMAE encoder [S1, unofficial].
2.  **Generate Modulating Parameters:**
    *   Pass $\mathbf{z}_l$ through Mapping Network 1 to generate parameters $(\gamma, \beta)$ for the first few layers of the Neural-PIL.
    *   Pass roughness $b_r$ through Mapping Network 2 to generate parameters $(\gamma, \beta)$ for the penultimate layer [S1, unofficial].
3.  **Forward Pass Neural-PIL:**
    *   Input $\omega_r$ into the first layer.
    *   Apply FiLM-SIREN layers. For the first few layers, use parameters from Mapping Network 1. For the penultimate layer, use parameters from Mapping Network 2.
    *   Output $\tilde{L}_i(\omega_r, b_r)$ [S1, unofficial].
4.  **Render:** Use $\tilde{L}_i(\omega_r, b_r)$ in the radiance equation to compute pixel color [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FiLMSIREN(nn.Module):
    """FiLM-SIREN layer: phi(y) = sin(lambda0 * (gamma * y) + beta)"""
    def __init__(self, in_features, out_features):
        super().__init__()
        self.linear = nn.Linear(in_features, out_features)
        # Modulating parameters are provided at forward time from mapping networks
        # lambda0 is a scalar, beta and gamma are vectors of size out_features

    def forward(self, x, lam0, beta, gamma):
        y = self.linear(x)
        # Hadamard product gamma * y
        modulated = lam0 * (gamma * y) + beta
        return torch.sin(modulated)

class MappingNetwork(nn.Module):
    """Predicts modulating parameters from conditioning vector"""
    def __init__(self, cond_dim, layer_dim, num_layers):
        super().__init__()
        self.layers = nn.ModuleList()
        for i in range(num_layers):
            self.layers.append(nn.Linear(cond_dim if i == 0 else layer_dim, layer_dim))
            self.layers.append(nn.ReLU())
        self.out_layer = nn.Linear(layer_dim, 3 * layer_dim) # Outputs lambda0, beta, gamma

    def forward(self, x):
        for layer in self.layers:
            x = layer(x)
        params = self.out_layer(x)
        # Split into lambda0 (scalar per sample, but here treated as broadcastable or per-feature)
        # Note: Source says lambda0 is scalar, beta/gamma are vectors.
        # Implementation detail: lambda0 might be a single scalar or per-feature.
        # Assuming per-feature for stability in FiLM, or scalar broadcast.
        # Source: "scalar lambda0 and vectors beta and gamma"
        # We reshape to match layer_dim.
        lam0 = params[:, :1] # Scalar per sample, shape (B, 1)
        beta = params[:, 1:layer_dim+1]
        gamma = params[:, layer_dim+1:]
        return lam0, beta, gamma

class NeuralPIL(nn.Module):
    """
    Neural Pre-Integrated Lighting
    Inputs: z_l (128), omega_r (3), b_r (scalar)
    Output: L_tilde (3, RGB)
    """
    def __init__(self, z_dim=128, hidden_dim=256, num_hidden_layers=4):
        super().__init__()
        self.z_dim = z_dim
        self.num_hidden_layers = num_hidden_layers
        
        # Mapping Networks
        # Map 1: z_l -> params for first few layers
        self.map1 = MappingNetwork(z_dim, hidden_dim, num_layers=2)
        # Map 2: b_r -> params for penultimate layer
        # b_r is scalar, so cond_dim=1
        self.map2 = MappingNetwork(1, hidden_dim, num_layers=2)
        
        # Neural-PIL Layers
        # Input dim: 3 (omega_r)
        # First layer uses map1 params
        self.layers = nn.ModuleList()
        
        # First layer
        self.layers.append(FiLMSIREN(3, hidden_dim))
        
        # Hidden layers
        for _ in range(num_hidden_layers - 2):
            self.layers.append(FiLMSIREN(hidden_dim, hidden_dim))
            
        # Penultimate layer (uses map2 params)
        self.layers.append(FiLMSIREN(hidden_dim, hidden_dim))
        
        # Final layer (standard SIREN or linear? Source implies MLP output)
        # Source says "predicts pre-integrated light". Usually final layer is linear or sin.
        # Assuming standard linear output for RGB, or sin if consistent.
        # Given "MLP that directly produces", and SIREN context, often final is linear.
        self.final_layer = nn.Linear(hidden_dim, 3)

    def forward(self, z_l, omega_r, b_r):
        # z_l: (B, 128)
        # omega_r: (B, 3)
        # b_r: (B,)
        
        # Get params for first layers from z_l
        lam0_1, beta_1, gamma_1 = self.map1(z_l)
        
        # Get params for penultimate layer from b_r
        lam0_2, beta_2, gamma_2 = self.map2(b_r.unsqueeze(1))
        
        x = omega_r
        
        # First layer: conditioned by z_l
        x = self.layers[0](x, lam0_1, beta_1, gamma_1)
        
        # Intermediate layers: no conditioning mentioned, assume standard SIREN or pass identity params?
        # Source: "condition the first few layers... condition a later layer...".
        # Implies middle layers are unconditioned or use default params.
        # Default SIREN: sin(Wx + b). We can set lam0=1, beta=0, gamma=1 for unconditioned.
        lam0_default = torch.ones_like(lam0_1)
        beta_default = torch.zeros_like(beta_1)
        gamma_default = torch.ones_like(gamma_1)
        
        for i in range(1, self.num_hidden_layers - 1):
            x = self.layers[i](x, lam0_default, beta_default, gamma_default)
            
        # Penultimate layer: conditioned by b_r
        x = self.layers[self.num_hidden_layers - 1](x, lam0_2, beta_2, gamma_2)
        
        # Final output
        out = self.final_layer(x)
        return out
```

**Edge Cases and Pitfalls**
*   **Precision:** The use of sine activations requires careful initialization (SIREN initialization) to avoid vanishing gradients [S1, unofficial].
*   **Boundary Conditions:** The roughness $b_r$ is a scalar; ensure it is clamped to valid BRDF roughness ranges (e.g., [0, 1]) to prevent out-of-distribution queries [G].
*   **Memory:** Training Neural-PIL jointly with other networks is memory-intensive; the source recommends freezing the Light-SMAE encoder [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Inference is $O(1)$ relative to the number of light samples, as it replaces numerical integration with a single network forward pass [S1, unofficial].
*   **Memory Complexity:** Depends on the size of the Neural-PIL MLP and the two mapping networks. Significantly lower than storing high-resolution environment maps or high-band SH coefficients [S1, unofficial].
*   **Strengths:** Captures fine illumination details that SG/SH representations miss without the associated memory and speed penalties [S1, unofficial].
*   **Limitations:** Requires training a separate autoencoder (Light-SMAE) for the illumination embedding [S1, unofficial].
*   **When to use:** When high-fidelity reflectance decomposition is needed and differentiable rendering frameworks lack native support for fine-detail light integration [S1, unofficial].

## Background
Neural-PIL was introduced in the paper "Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition" [S1, unofficial]. It addresses the limitation of standard differentiable rendering frameworks that rely on SG or SH light representations, which lack fine details or incur high memory costs [S1, unofficial]. The method leverages a pi-GAN-like architecture with FiLM-SIREN layers [S1, unofficial].

## Sources
[S1] Neural-PIL: Neural Pre-Integrated Lighting for Reflectance Decomposition, https://doi.org/10.48550/arxiv.2110.14373, fetched 2026-10-26 (unofficial)
