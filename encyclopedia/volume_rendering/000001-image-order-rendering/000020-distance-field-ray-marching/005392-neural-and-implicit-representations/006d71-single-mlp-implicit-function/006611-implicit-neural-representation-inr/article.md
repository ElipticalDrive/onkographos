# Implicit Neural Representation (INR)

Also called: Continuous volume representation, Hashgrid INR, Implicit Neural Representation (INR) for CT Template, MLP INR, MLP-based LAC estimation, Network Architecture for INR

## Overview
Implicit Neural Representations (INRs) encode volumetric or geometric data as the output of a continuous function, typically a Multi-Layer Perceptron (MLP), mapping 3D coordinates to physical properties like density or signed distance [S1, S2]. This method solves the problem of discretization artifacts and storage overhead associated with voxel grids by providing a differentiable representation suitable for volume rendering and tomography [S1, S2]. Unlike discrete voxel-based methods, INRs integrate seamlessly with gradient-based optimization for reconstruction from sparse or limited view projections [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D coordinates $\mathbf{v} = (x, y, z)$ sampled from the scene domain, typically normalized to $[-1, 1]^3$ [S1].
*   **Output:** A scalar value representing the physical property (e.g., Linear Attenuation Coefficient (LAC) for CT, or Signed Distance Function (SDF) for geometry) [S1, S2].

**Data Structures:**
*   **MLP Parameters:** A set of weights $\mathbf{W}_i$ and biases $\mathbf{b}_i$ for each layer $i$ [S2].
*   **Encoding Parameters:** For Fourier-based methods, a random vector $\mathbf{B}$ and bandwidth $\kappa$ [S1]. For hashgrid methods, hash tables mapping coordinates to feature vectors to accelerate computation [S2].

**Algorithm Steps:**

1.  **Coordinate Sampling:**
    Sample a grid of coordinates $\mathbf{V} \in \mathbb{R}^{\beta \times 3}$ from the scene boundaries $[-1, 1]^3$ [S1]. To enforce continuity and prevent the network from memorizing discrete grid points, perturb each coordinate randomly within its respective voxel [S1].
    *   *Parameter:* Voxel grid resolution $\beta$. Typical value: $\beta = 80$, resulting in $80^3$ sampled coordinates per iteration [S1].

2.  **Positional Encoding:**
    Apply an encoding function $\gamma(\mathbf{v})$ to the input coordinates before the first MLP layer to improve high-frequency fitting [S1, S2].
    *   **Gaussian Random Fourier Features (GRFF):**
        $$ \gamma(\mathbf{v}) = [\cos(2\pi \kappa \mathbf{B} \mathbf{v}), \sin(2\pi \kappa \mathbf{B} \mathbf{v})] $$
        Where $\mathbf{v}=(x,y,z)$, $\mathbf{B}$ is a vector randomly sampled from $\mathcal{N}(0, I)$, and operations are element-wise [S1].
        *   *Parameter:* Bandwidth factor $\kappa$. Controls the sharpness of the output. If $\kappa$ is too low, high-frequency content is lost; if too high, the network fits spurious noise in the measurements (e.g., sinogram) [S1].
    *   **Hashgrid Encoding:**
        Maps coordinates to dense feature vectors via hash tables, significantly speeding up training and rendering compared to Fourier features [S2].
    *   **Fourier Feature Mapping (ReLU variant):**
        Projects the input onto a list of sines and cosines in the first layer, followed by ReLU activations in subsequent layers [S2].

3.  **Network Forward Pass:**
    Compute the network output $f(\mathbf{p})$ using the MLP composition:
    $$ f(\mathbf{p}) = \mathbf{W}_n \circ f_{n-1} \circ \dots \circ f_0(\mathbf{p}) + \mathbf{b}_n $$
    Where $f_i(\mathbf{p}_i) = \phi(\mathbf{W}_i \mathbf{p}_i + \mathbf{b}_i)$ [S2].
    *   *Architecture:* Typically 4 fully connected layers [S1].
    *   *Activation Function $\phi$:*
        *   **Sine ($\phi = \sin$):** Used for powerful INR architectures in surface reconstruction [S2].
        *   **ReLU:** Used with Fourier feature mapping for image-based reconstruction [S2].

4.  **Loss Computation and Optimization:**
    Compute the loss between the network output and the target data (e.g., measured sinogram values or SDF ground truth) and optimize the MLP parameters $\theta$ using gradient descent [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sample_coordinates(beta, bounds=(-1.0, 1.0)):
    # Generate beta^3 linearly spaced coordinates in [-1, 1]^3
    coords = np.linspace(bounds[0], bounds[1], beta)
    grid = np.meshgrid(coords, coords, coords)
    V = np.stack(grid, axis=-1).reshape(-1, 3)
    # Perturb randomly within voxels to enforce continuity
    perturbation = np.random.uniform(0, 1.0/beta, V.shape)
    return V + perturbation

def grff_encode(v, kappa, B):
    # v: (N, 3), B: (3,), element-wise multiplication per evidence
    return np.concatenate([np.cos(2 * np.pi * kappa * B * v), 
                           np.sin(2 * np.pi * kappa * B * v)], axis=-1)

def mlp_forward(p, weights, biases, activation):
    x = p
    for W, b in zip(weights, biases[:-1]):
        x = activation(W @ x + b)
    return weights[-1] @ x + biases[-1]

def train_inr(V, targets, weights, biases, kappa, B, activation, lr, epochs):
    for epoch in range(epochs):
        encoded = grff_encode(V, kappa, B)
        pred = mlp_forward(encoded, weights, biases, activation)
        loss = np.mean((pred - targets) ** 2)
        # Gradient descent step (conceptual)
        grad = np.gradient(loss, weights, biases)
        weights -= lr * grad[0]
        biases -= lr * grad[1]
    return weights, biases
```

**Pitfalls:**
*   **Bandwidth Selection:** Setting $\kappa$ in GRFF incorrectly leads to either underfitting (low $\kappa$) or overfitting to noise (high $\kappa$) [S1].
*   **Training Speed:** Standard Fourier feature MLPs train slowly; hashgrid encodings are recommended for speed [S2].
*   **Misalignment:** In tomography, uncorrected tilt-axis errors cause streaking artifacts. The INR must be trained with pose optimization or pre-aligned data [S3, unofficial].

## Subtopics

### [VC-INR Meta-Training](19798-vc-inr-meta-training.md)
Pre-training phase for Variational Compression of Implicit Neural Representations (VC-INR). Learns a shared network parameterization and latent modulation initialization for rapid adaptation to new data instances with minimal storage overhead. Sets itself apart by optimizing the initialization itself via meta-learning rather than training weights for a single instance. Choose when rapid adaptation to new data with minimal storage is required.

### [WIRE (Wavelet Implicit Neural Representations)](20126-wire-wavelet-implicit-neural-representations.md)
Uses a continuous complex Gabor wavelet as the activation nonlinearity to achieve high accuracy, fast training, and robustness to noise by leveraging optimal space-frequency compactness. Sets itself apart by combining Gaussian spatial compactness with complex exponential frequency compactness. Choose when high-frequency fitting and noise robustness are prioritized over standard activations.

### [SPW (Semantic Priors in Weights)](20854-spw-semantic-priors-in-weights.md)
Reparameterization method that encodes semantic priors into the weights of an INR using a pre-trained Semantic Neural Network (SNN) and Weight Generation Network (WGN) to enhance representational capacity without increasing inference cost. Sets itself apart by decoupling semantic feature extraction from implicit function learning. Choose when injecting specific semantic context into weights is needed for a target signal.

### [SPW (Semantic Priors in Weights)](20854-spw-semantic-priors-in-weights.md)
A technique where the full 3D orientation of the volume corresponding to each tilt image is directly optimized during the INR training process to correct for tilt axis errors and misalignment. Sets itself apart by jointly optimizing network weights and pose parameters rather than assuming fixed poses. Choose when dealing with electron tomography data subject to rotational misalignment. Large misalignment increases convergence epochs, so coarse preprocessing is recommended.

## Operating characteristics
*   **Time Complexity:** Training time is generally slower than traditional algebraic reconstruction techniques (ART/SIRT) for a single instance due to the iterative nature of neural optimization [S1, S2]. Hashgrid-based INRs significantly reduce training and rendering times compared to Fourier-feature-based INRs [S2].
*   **Memory Complexity:** The memory footprint is determined by the MLP weights and encoding tables [S1, S2].
*   **Performance:** INRs can achieve high Structural Similarity (SSIM) (e.g., average SSIM of 0.974) even with limited views or significant misalignment when combined with pose optimization [S3, unofficial].
*   **Strengths:** Differentiable nature allows for gradient-based optimization, compact storage, and robustness to noise when properly regularized [S1, S2].
*   **Limitations:** Slow convergence for high-resolution details without advanced encodings; sensitive to hyperparameters like $\kappa$ [S1]; requires careful handling of misalignment in tomographic applications [S3, unofficial].
*   **When to use:** When dealing with sparse or limited view data, or when integrating geometric priors into the reconstruction process [S1, S2].
*   **When not to use:** When real-time inference on low-power hardware is critical without pre-trained models, or when the scene is static and high-resolution voxel storage is not a constraint [S2].

## Background
Implicit Neural Representations originated from the need to represent continuous signals and geometric shapes without discretization [G]. The use of MLPs to approximate continuous functions for graphics was popularized by works such as SIREN (Sinusoidal Representation Networks) [S2]. In the context of volume rendering and tomography, INRs have been applied to reconstruct dynamic scenes from limited views using parametric motion fields [S1] and to inpaint missing wedges in electron tomography while jointly optimizing alignment [S3, unofficial]. The method builds on [Single MLP Implicit Function](21357-single-mlp-implicit-function.md) by introducing specific encoding strategies (Fourier, Hashgrid) and training protocols (meta-training, pose optimization) to enhance performance and efficiency [G].

## Sources
[S1] Dynamic CT Reconstruction from Limited Views with Implicit Neural Representations and Parametric Motion Fields, https://doi.org/10.1109/iccv48922.2021.00226, fetched 2026-10-26
[S2] Geometric implicit neural representations for signed distance functions, https://doi.org/10.1016/j.cag.2024.104085, fetched 2026-10-26
[S3] Missing Wedge Inpainting and Joint Alignment in Electron Tomography through Implicit Neural Representations, https://arxiv.org/abs/2512.08113, fetched 2026-10-27 (unofficial)
