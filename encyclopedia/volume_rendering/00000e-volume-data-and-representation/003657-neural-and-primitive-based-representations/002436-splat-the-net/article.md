# Splat the Net

Also called: Neural Primitives, Splat the Net: Radiance Fields with Splattable Neural Primitives

## Overview
Splat the Net is a neural primitive-based volume rendering method that replaces Gaussian primitives with splattable neural density fields to represent 3D scenes [S1, unofficial]. It solves the trade-off between representation expressivity and rendering speed by using MLP-based primitives that require roughly 10× fewer primitives than 3DGS for the same scene representation while maintaining >100 FPS rendering speed [S1, unofficial]. Unlike its sibling [Gaussian Head Avatar (GHA)](1478-gaussian-head-avatar-gha.md) which uses fixed Gaussian kernels, Splat the Net uses an 8-neuron MLP to define the density profile of each primitive, offering greater expressivity under identical memory budgets [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial position $\mathbf{x}$, viewing direction $\mathbf{d}$, and optionally time $\xi_t$ or light direction $\mathbf{d}_l$.
*   **Output:** Density $\sigma$ and color $\mathbf{c}$ for a ray passing through the scene.

**Data Structures:**
Each primitive $i$ stores the following parameters (total 99 parameters per primitive):
1.  **Geometry (10 parameters):**
    *   Mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$.
    *   Scale $\mathbf{s}_i \in \mathbb{R}^3$.
    *   Rotation quaternion $\mathbf{q}_i \in \mathbb{R}^4$.
2.  **Neural Density Field (41 parameters):**
    *   An 8-neuron Multi-Layer Perceptron (MLP) defining the radial density profile.
    *   Weights $W_1, W_2$ and biases $b_1, b_2$.
    *   For dynamic scenes, an additional temporal weight $W_t \in \mathbb{R}^{N_\sigma}$ is included.
3.  **Appearance (48 parameters):**
    *   Spherical Harmonic (SH) coefficients for color.

**Numbered Steps:**

1.  **Initialization:** Uniformly sample $N$ primitives within the scene's bounding box. For dynamic scenes, $N=100,000$ [S1, unofficial].
2.  **Density Evaluation:** For a query point $\mathbf{x}$ relative to primitive $i$, compute the local coordinate $\mathbf{x}' = \mathbf{R}_i^T (\mathbf{x} - \boldsymbol{\mu}_i)$, where $\mathbf{R}_i$ is the rotation matrix derived from $\mathbf{q}_i$. Normalize by scale $\mathbf{s}_i$. Pass the normalized distance through the MLP to get density $\sigma_i(\mathbf{x})$.
    *   Static density formula:
        $$ \sigma_i(\mathbf{x}) = \text{MLP}(\|\mathbf{x}'\|) $$
    *   Dynamic density formula (Eulerian approach):
        $$ f_*(\mathbf{x}, \xi_t) = W_2 \cos(\omega_0 \cdot W_1(\mathbf{x}) + \xi_t \cdot W_t + \mathbf{b}_1) + \mathbf{b}_2 $$
        where $\xi_t \in [0,1]$ is the temporal variable and $W_t$ is a learnable temporal weight [S1, unofficial].
3.  **Color Evaluation:**
    *   **Static/Novel View Synthesis:** Color is determined by the zero-order SH coefficient of the primitive, modulated by the view direction $\mathbf{d}$.
    *   **Dynamic:** The zero-order SH coefficient $S(\xi_t)$ is modeled as a function of time:
        $$ S(\xi_t) = S_0 + P_n(\xi_t) + F_l(\xi_t) $$
        where $P_n(\xi_t) = \sum_{i=1}^n a_i \xi_t^i$ is a polynomial of degree $n$, and $F_l(\xi_t)$ is a Fourier series of order $l$. In experiments, $n=4$ and $l=4$ [S1, unofficial].
    *   **Relighting:** Color is formulated as:
        $$ \mathbf{c} = \mathbf{c}_{dc} + \mathbf{c}(\mathbf{x}, \mathbf{d}, \mathbf{d}_l) $$
        where $\mathbf{c}_{dc}$ is a constant color and $\mathbf{c}(\cdot)$ is a neural color function dependent on position, view direction, and light direction $\mathbf{d}_l$ [S1, unofficial].
4.  **Rendering:** Perform volumetric rendering (e.g., alpha compositing) by sampling points along rays, evaluating $\sigma_i$ and $\mathbf{c}_i$ for all primitives, and accumulating opacity and color.
5.  **Population Control (Densification and Pruning):**
    *   **Densification:** Split or clone primitives based on gradient magnitude.
    *   **Pruning:** Remove primitives with low opacity or large scale.
    *   **Thresholds:**
        *   Growing scale threshold: $10^{-2}$ [S1, unofficial].
        *   Pruning scale threshold: $0.5$ [S1, unofficial].
        *   MLP-gradient-based densification threshold: $10^{-4}$ (Mip-NeRF360/Tanks&Temples/Deep Blending), $10^{-5}$ (BlenderNeRF) [S1, unofficial].
        *   MLP-gradient-based pruning threshold: $2 \times 10^{-6}$ (Mip-NeRF360/Tanks&Temples/Deep Blending), $10^{-6}$ (BlenderNeRF) [S1, unofficial].
    *   **Frequency:**
        *   Mip-NeRF360/Tanks&Temples/Deep Blending: Every 500 iterations between 1k and 15k [S1, unofficial].
        *   BlenderNeRF: Every 200 iterations between 1k and 20k [S1, unofficial].
        *   Dynamic scenes: Every 500 iterations between 1k and 30k [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def evaluate_neural_primitive_density(x_local, params):
    """
    Evaluates the density of a single neural primitive at a local coordinate.
    x_local: normalized distance from center (scalar or vector magnitude)
    params: dict containing W1, b1, W2, b2, Wt (optional), xi_t (optional)
    """
    # Static part
    z1 = np.dot(params['W1'], x_local) + params['b1']
    # Activation is implicitly handled by the specific MLP architecture described
    # Based on formula: W2 cos(omega0 * W1(x) + xi_t * Wt + b1) + b2
    # Assuming W1 is a weight matrix/vector and x_local is the input vector
    # Note: The source formula implies a specific 8-neuron MLP structure.
    # Simplified implementation based on the provided formula:
    
    # If dynamic
    if 'Wt' in params and 'xi_t' in params:
        temporal_term = np.dot(params['Wt'], params['xi_t'])
    else:
        temporal_term = 0.0
        
    # The formula uses cos, implying a specific activation or layer structure.
    # We implement the exact formula provided for f_*(x, xi_t)
    # However, the general MLP is described as 8-neuron. 
    # The formula f_* seems to be a specific instance or the core density function.
    # We will use the explicit formula for density evaluation as given in the source.
    
    # Re-interpreting the formula for general MLP:
    # The source states "41 parameters from 8-neuron MLP".
    # The formula f_* is given for the temporal density field.
    # For static, it is likely a similar MLP structure.
    # We assume the provided formula is the core density computation.
    
    # Let's implement the formula exactly as stated for the density field
    # f_*(x, xi_t) = W2 cos(omega0 * W1(x) + xi_t * Wt + b1) + b2
    # This looks like a single layer or a specific activation. 
    # Given the ambiguity of "8-neuron MLP" vs the specific formula, 
    # we stick to the formula for the density value.
    
    # Note: omega0 is likely a constant or parameter. Not explicitly defined as variable.
    # Assuming omega0 is part of W1 or a constant. 
    # The source says "W1(x)" so W1 is applied to x.
    
    # To be safe and implementable:
    # We assume the MLP is a standard feedforward network.
    # But the formula is explicit. We will use the explicit formula.
    
    # If x_local is a vector, W1(x) is a dot product or matrix mult.
    # Let's assume W1 is a vector and x_local is a scalar distance for radial symmetry?
    # Or W1 is a matrix.
    # Given "8-neuron", it's likely a small MLP.
    # We will implement the formula as a black box function if the structure is unclear,
    # but the formula is explicit.
    
    # Let's assume the formula is the density function.
    val = params['W2'] * np.cos(params['omega0'] * np.dot(params['W1'], x_local) + temporal_term + params['b1']) + params['b2']
    return np.maximum(val, 0.0) # Density must be non-negative

def render_ray(ray_origin, ray_direction, primitives, camera_params):
    """
    Renders a single ray using splatting of neural primitives.
    """
    # 1. Intersect ray with bounding box of each primitive or use global bounding box
    # 2. Sample points along ray
    # 3. For each point, evaluate density and color from all primitives
    # 4. Composite
    
    # This is a high-level description as the exact splatting kernel for neural primitives
    # is not fully detailed in the text beyond the density formula.
    # The key is that primitives are "splattable", implying a kernel-based approach
    # where the neural network defines the kernel shape.
    
    pass
```

**Parameters:**
*   MLP learning rate: $10^{-3}$ [S1, unofficial].
*   Primitive means learning rate: $1.6 \times 10^{-4}$ [S1, unofficial].
*   Primitive scales learning rate: $5 \times 10^{-3}$ [S1, unofficial].
*   Primitive quaternions learning rate: $10^{-3}$ [S1, unofficial].
*   SH coefficients learning rate: $2.5 \times 10^{-3}$ [S1, unofficial].
*   Polynomial degree $n$ and Fourier series order $l$: 4 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The method requires higher FLOPs per integration (769 FLOPs) compared to 3DGS (43 FLOPs) [S1, unofficial]. Implementation must optimize the MLP evaluation to maintain real-time performance.
*   **Boundaries:** Densification and pruning thresholds vary by dataset (BlenderNeRF vs. Mip-NeRF360/etc.). Using incorrect thresholds can lead to over-sampling or under-representation [S1, unofficial].
*   **Degenerate Input:** The temporal variable $\xi_t$ must be normalized to $[0,1]$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Higher computational cost per primitive than Gaussian primitives (769 FLOPs vs 43 FLOPs) [S1, unofficial]. However, due to requiring 10× fewer primitives, overall inference speed remains >100 FPS, comparable to 3DGS [S1, unofficial].
*   **Memory Complexity:** Lower memory usage than 3DGS for equivalent quality. Requires half the number of primitives for the same memory budget [S1, unofficial].
*   **Strengths:** Higher expressivity than Gaussian primitives under the same memory budget; real-time rendering speed; supports dynamic scenes and relighting [S1, unofficial].
*   **Limitations:** Higher per-primitive computational cost; requires careful tuning of densification/pruning thresholds [S1, unofficial].
*   **When to use:** When memory is constrained but high fidelity is required, and real-time performance is needed.
*   **When not to use:** When maximum possible rendering speed is required regardless of quality, or when scene complexity is low enough that fewer primitives are not needed.

## Background
Splat the Net was introduced in the paper "Splat the Net: Radiance Fields with Splattable Neural Primitives" [S1, unofficial]. It builds on the principles of neural and primitive-based representations [G]. The method was developed to address the limitations of Gaussian primitives in 3DGS, specifically their lower expressivity per parameter [S1, unofficial]. It is used for novel view synthesis, dynamic scene reconstruction, and relighting [S1, unofficial].

## Sources
[S1] Splat the Net: Radiance Fields with Splattable Neural Primitives, https://doi.org/10.48550/arxiv.2510.08491, fetched 2026-10-26 (unofficial)
