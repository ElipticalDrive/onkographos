# MLP Model (Density Baseline)
Also called: positional encoder + MLP

## Overview
A baseline neural volume representation that maps 3D Cartesian positions to density using a Multi-Layer Perceptron (MLP) with explicit positional encoding. It serves as a reference for density reconstruction capabilities in neural tomography, distinct from hash-grid or latent-feature approaches.

## How it works
**Inputs and Outputs:**
- **Input:** A 3D Cartesian position vector $\mathbf{x} \in \mathbb{R}^3$, typically mapped to an Axis-Aligned Bounding Box (AABB) [S1, unofficial].
- **Output:** A single scalar representing the base-10 logarithm of the number density, $\log_{10} n_e$ [S1, unofficial].

**Data Structures and Layout:**
The input to the MLP is a concatenated vector of dimension 48, formed by:
1.  **Raw Position:** 3 dimensions.
2.  **Fourier Positional Encoding:** Axis-wise Fourier features with $L=6$ bands. Frequencies are $2^0, \dots, 2^5$. For each axis, this generates $\sin(2^k \pi x)$ and $\cos(2^k \pi x)$ for $k \in \{0, \dots, 5\}$. Total dimensions: $2 \text{ (sin/cos)} \times 3 \text{ (axes)} \times 6 \text{ (bands)} = 36$ dimensions [S1, unofficial].
3.  **Spherical Harmonic Encoding:** Degree 2 spherical harmonics of the unit position direction. Total dimensions: $(2+1)^2 = 9$ dimensions [S1, unofficial].

**MLP Architecture:**
- **Hidden Layers:** 4 layers, each with width 256.
- **Activation:** SiLU (Sigmoid Linear Unit) for hidden layers [S1, unofficial].
- **Output Layer:** Linear layer producing a single scalar.
- **Initialization:** The final layer bias is initialized to the midpoint of the configured $\log n_e$ clamp range [S1, unofficial].
- **Total Parameters:** 210,177 [S1, unofficial].

**Step-by-Step Method:**
1.  Normalize or map the input 3D position $\mathbf{x}$ to the AABB.
2.  Compute the Fourier positional encoding $\mathbf{f}_{fourier}(\mathbf{x})$ using the specified frequencies and axes.
3.  Compute the spherical harmonic encoding $\mathbf{f}_{sh}(\hat{\mathbf{x}})$ where $\hat{\mathbf{x}}$ is the normalized direction of $\mathbf{x}$.
4.  Concatenate the raw position, Fourier encoding, and SH encoding: $\mathbf{v} = [\mathbf{x}, \mathbf{f}_{fourier}(\mathbf{x}), \mathbf{f}_{sh}(\hat{\mathbf{x}})]$.
5.  Pass $\mathbf{v}$ through the MLP:
    $$ \mathbf{h}_1 = \text{SiLU}(\mathbf{W}_1 \mathbf{v} + \mathbf{b}_1) $$
    $$ \mathbf{h}_2 = \text{SiLU}(\mathbf{W}_2 \mathbf{h}_1 + \mathbf{b}_2) $$
    $$ \mathbf{h}_3 = \text{SiLU}(\mathbf{W}_3 \mathbf{h}_2 + \mathbf{b}_3) $$
    $$ \mathbf{h}_4 = \text{SiLU}(\mathbf{W}_4 \mathbf{h}_3 + \mathbf{b}_4) $$
    $$ \log_{10} n_e = \mathbf{w}_5 \cdot \mathbf{h}_4 + b_5 $$
6.  Apply clamping to $\log_{10} n_e$ based on configured bounds (as per parent method [Parent](6361-learned-fields-prediction-with-clamping.md)).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def fourier_encoding(x, num_bands=6):
    """
    Computes axis-wise Fourier positional encoding.
    x: (3,) array
    Returns: (36,) array
    """
    freqs = np.array([2**k for k in range(num_bands)]) # 2^0 to 2^5
    # x is (3,), freqs is (6,)
    # We want sin(2 * pi * freq * x_axis) and cos(...)
    # Expand dims: x -> (1, 3), freqs -> (6, 1)
    # Result shape: (6, 3) for sin, (6, 3) for cos
    # Flatten to (36,)
    
    # Calculate arguments: 2 * pi * freq_k * x_i
    # Using broadcasting: (6, 1) * (1, 3) -> (6, 3)
    args = 2.0 * math.pi * np.outer(freqs, x)
    
    sin_vals = np.sin(args).flatten()
    cos_vals = np.cos(args).flatten()
    
    return np.concatenate([sin_vals, cos_vals])

def spherical_harmonic_degree_2(x):
    """
    Computes degree 2 spherical harmonics for the direction of x.
    x: (3,) array
    Returns: (9,) array
    """
    # Normalize x to get direction
    norm = np.linalg.norm(x)
    if norm == 0:
        return np.zeros(9)
    d = x / norm
    
    # Degree 2 SH basis functions (real spherical harmonics)
    # Y_0^0
    y00 = 0.5 * math.sqrt(1.0 / math.pi)
    
    # Y_1^-1, Y_1^0, Y_1^1
    y1m1 = math.sqrt(3.0 / (4.0 * math.pi)) * d[1] # -y
    y10 = math.sqrt(3.0 / (4.0 * math.pi)) * d[2]  # z
    y11 = math.sqrt(3.0 / (4.0 * math.pi)) * d[0]  # x
    
    # Y_2^-2, Y_2^-1, Y_2^0, Y_2^1, Y_2^2
    # Standard real SH definitions
    y2m2 = math.sqrt(15.0 / math.pi) * d[0] * d[1]
    y2m1 = math.sqrt(15.0 / math.pi) * d[1] * d[2]
    y20 = math.sqrt(5.0 / (16.0 * math.pi)) * (3 * d[2]**2 - 1)
    y21 = math.sqrt(15.0 / math.pi) * d[2] * d[0]
    y22 = math.sqrt(15.0 / math.pi) * (d[0]**2 - d[1]**2)
    
    return np.array([y00, y1m1, y10, y11, y2m2, y2m1, y20, y21, y22])

def mlp_model_forward(x, weights, biases):
    """
    Forward pass of the MLP baseline.
    x: (3,) raw position
    weights: list of 5 weight matrices [W1, W2, W3, W4, W5]
    biases: list of 5 bias vectors [b1, b2, b3, b4, b5]
    Returns: scalar log10_ne
    """
    # 1. Encode
    raw_pos = x
    fourier_feat = fourier_encoding(x)
    sh_feat = spherical_harmonic_degree_2(x)
    
    # 2. Concatenate
    input_vec = np.concatenate([raw_pos, fourier_feat, sh_feat]) # (48,)
    
    # 3. MLP Layers
    h = input_vec
    for i in range(4):
        # Linear + SiLU
        h = np.dot(h, weights[i]) + biases[i]
        h = h / (1 + np.exp(-h)) # SiLU approximation or explicit SiLU function
        
    # 4. Output Layer
    log10_ne = np.dot(h, weights[4]) + biases[4]
    
    return log10_ne
```

**Parameters:**
- Fourier bands $L=6$ [S1, unofficial].
- MLP hidden width: 256 [S1, unofficial].
- MLP hidden layers: 4 [S1, unofficial].
- Final layer bias initialization: Midpoint of $\log n_e$ clamp range [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero in spherical harmonic calculation if $\mathbf{x} = \mathbf{0}$ [G].
- Precision issues with high-frequency Fourier features if positions are not normalized to the AABB [S1, unofficial].

## Subtopics

### HeadGaS Baseline (Direct Input MLP)
A variant where the expression vector $e_i$ and encoded position $\boldsymbol{\mu}$ are fed directly into an MLP $\boldsymbol{\rho}(\boldsymbol{\mu})$ to predict color and opacity, without a basis of latent features [S2]. It uses 5 linear layers with leaky ReLU activations [S2]. On the INSTA dataset with 295k Gaussians, it achieves 32.08 PSNR with 0.010s rendering time [S2]. It is slower than the proposed HeadGaS method despite lacking MLP computation during rendering, due to the significantly larger number of Gaussians (295k vs 28k) [S2].

## Operating characteristics
- **Complexity:** The model has 210,177 parameters [S1, unofficial]. The forward pass involves fixed-size matrix multiplications and trigonometric evaluations, making it independent of scene complexity (number of objects) but dependent on query frequency.
- **Performance:** Not given in the sources for this specific baseline in isolation, but compared to HeadGaS (which uses 28k Gaussians and 0.004s rendering), the HeadGaS Baseline (295k Gaussians) takes 0.010s [S2]. The MLP density baseline is used in neural tomography contexts where field fidelity is prioritized [S1, unofficial].
- **Strengths:** Simple architecture, explicit control over frequency bands via positional encoding.
- **Limitations:** High parameter count relative to simple functions; requires careful initialization of the final bias [S1, unofficial].

## Background
Originated as a baseline in neural tomography research to evaluate density reconstruction capabilities using positional encoders and MLPs [S1, unofficial]. It contrasts with more complex representations like multiresolution hash grids [S1, unofficial].

## Sources
[S1] Image Fidelity is Not Field Fidelity: Joint Thermodynamic Reconstruction and Error Localization in Neural Tomography, https://doi.org/10.48550/arxiv.2609.28868, fetched 2026-10-26 (unofficial)
[S2] HeadGaS: Real-Time Animatable Head Avatars via 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-72627-9_26, fetched 2026-10-27
