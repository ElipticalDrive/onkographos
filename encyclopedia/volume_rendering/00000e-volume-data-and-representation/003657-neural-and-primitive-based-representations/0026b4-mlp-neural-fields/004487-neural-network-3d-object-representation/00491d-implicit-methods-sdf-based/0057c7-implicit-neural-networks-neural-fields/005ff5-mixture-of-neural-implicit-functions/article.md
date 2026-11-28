# Mixture of Neural Implicit Functions

Also called: Generative Neural Fields, Generative Neural Fields by Mixtures of Neural Implicit Functions, Generative Neural Fields with a Mixture of Neural Implicit Functions, mINF, mNIF

## Overview
Mixture of Neural Implicit Functions (mNIF) is a generative neural field method that represents data instances as a linear combination (weighted average) of multiple implicit basis networks, known as Neural Implicit Functions (NIFs) [S1]. It separates shared structural information into the basis networks and instance-specific details into latent mixture coefficients [S1]. Unlike HyperNet, which predicts weights directly and suffers from instability, or FiLM, which is restrictive and adds parameters, mNIF computes a weighted sum of basis network outputs, offering a flexible yet stable modulation scheme [S1]. It allows increasing model capacity by learning more neural basis functions while keeping inference compact via weighted model averaging [S1].

## How it works
### Inputs and Outputs
*   **Inputs:**
    *   Query coordinates $x \in \mathbb{R}^d$ (e.g., pixel coordinates for images, 3D coordinates for voxels/NeRF) [S1].
    *   Latent vector $\phi \in \mathbb{R}^H$ defining the instance [S1].
*   **Outputs:**
    *   Target values $y \in \mathbb{R}^k$ (e.g., RGB values, voxel densities) [S1].

### Data Structures
The model consists of $M$ basis networks and a projection mechanism for mixture coefficients.

1.  **Basis Networks:** A set of $M$ NIFs. The $m$-th basis network has parameters $\theta^m$.
    *   $\theta^m \triangleq \{W_m^{(0)}, b_m^{(0)}, \ldots, W_m^{(L+1)}, b_m^{(L+1)}\}$ [S1].
    *   $W_m^{(i)} \in \mathbb{R}^{W \times W}$ (or $\mathbb{R}^{W \times d}$ for input, $\mathbb{R}^{k \times W}$ for output) [S1].
    *   $b_m^{(i)} \in \mathbb{R}^W$ (or $\mathbb{R}^k$ for output) [S1].
    *   $L$: Number of hidden layers [S1].
    *   $W$: Hidden layer width [S1].
    *   $M$: Number of mixture components (basis networks) [S1].

2.  **Projection Matrix:** A matrix $T$ used to derive layer-specific mixture coefficients from the latent vector.
    *   $T$ maps $\phi \in \mathbb{R}^H$ to the vector of all mixture coefficients for all layers [S1].
    *   The total number of mixture coefficients is $M \times (L+1)$ (assuming coefficients for all layers including input/output or just hidden, the source implies layer-wise coefficients for the modulation step) [S1]. Specifically, Eq. 10 projects $\phi$ to $\alpha_m^{(i)}$ for all $i \in \{0, \dots, L+1\}$ and $m \in \{1, \dots, M\}$ [S1].

3.  **Latent Vector:** $\phi \in \mathbb{R}^H$, where $H$ is the dimensionality of the latent coefficient vector [S1].

### Numbered Steps

**Training Phase:**
1.  **Initialize:** Initialize $M$ basis networks with parameters $\theta^1, \ldots, \theta^M$ and the projection matrix $T$.
2.  **Optimize:** Optimize the basis parameters $\{\theta^1, \ldots, \theta^M\}$ and the projection matrix $T$ (and potentially latent vectors $\phi_j$ for each instance if using auto-decoding) using a signal reconstruction loss [S1].
    *   Two optimization strategies are supported:
        *   **Meta-learning:** Uses second-order gradient computation. Set the number of inner-loop iterations ($N_{inner}$) to 3 to balance performance and complexity [S1].
        *   **Auto-decoding:** Optimizes instance-specific latent vectors $\phi_j$ directly. This strategy surpasses meta-learning with first-order gradients in performance and is more efficient in speed and memory [S1].
3.  **Latent Prior (Optional for Generation):** Train a denoising diffusion probabilistic model to estimate the prior distribution $P_{latent}$ over the latent vectors $\phi$ [S1].

**Inference/Sampling Phase:**
1.  **Sample Latent:** Sample a context vector $\phi_j$ from the prior distribution $P_{latent}$ estimated by the diffusion model [S1].
2.  **Compute Coefficients:** Derive the mixture coefficients $\alpha_m^{(i)}$ for each layer $i$ and basis $m$ by projecting $\phi_j$ through the matrix $T$:
    $$ [\alpha_1^{(0)}, \ldots, \alpha_M^{(0)}, \alpha_1^{(1)}, \ldots, \alpha_M^{(L+1)}] = T \phi $$
    [S1]
3.  **Forward Pass:** For a query $x$, compute the output $y$ using the weighted mixture of basis networks.

### Formulas

**Standard INR Layer (Base):**
$$ h^{(i+1)} = \sigma(W^{(i)} h^{(i)} + b^{(i)}) $$
where $\sigma$ is an activation function [S1].

**SIREN Layer (Base):**
$$ h^{(i+1)} = \sin(w_0 (W^{(i)} h^{(i)} + b^{(i)})) $$
where $w_0$ is a scale hyperparameter [S1].

**mNIF Layer Operation (Core Modulation):**
The mNIF uses SIREN-style activations with weighted model averaging inside the activation:
$$ h^{(i+1)} = \sin\left(w_0 \sum_{m=1}^M \alpha_m^{(i)} g_m(h^{(i)})\right) $$
where $g_m(h^{(i)}) = W_m^{(i)} h^{(i)} + b_m^{(i)}$ is the linear transform of the $m$-th basis network at layer $i$ [S1].

Expanded form:
$$ h^{(i+1)} = \sin\left(w_0 \left(\sum_{m=1}^M \alpha_m^{(i)} W_m^{(i)} h^{(i)} + \sum_{m=1}^M \alpha_m^{(i)} b_m^{(i)}\right)\right) $$
[ S1 ]

**Full Network Forward Pass:**
$$ y = f_\theta(x) = f^{(L+1)} \circ \cdots \circ f^{(1)} \circ f^{(0)}(x) $$
where each $f^{(i)}$ is the mNIF layer operation defined above [S1].

### Parameters and Typical Values
*   $M$ (Number of mixtures): Performance saturates around $M = 256$. Increasing $M$ improves recall more than precision [S1].
*   $H$ (Latent dimension): Increasing $H$ improves reconstruction performance in both train and test splits due to increased degrees of freedom [S1].
*   $L$ (Hidden layer depth): Default $L=2$ for ablation studies, $L=5$ for longer training studies [S1].
*   $W$ (Hidden layer width): Default $W=64$ or $W=128$ [S1].
*   $w_0$: Scale hyperparameter for SIREN frequency [S1].
*   $N_{inner}$: Number of inner-loop iterations for meta-learning. Set to 3 [S1].

### Edge Cases and Pitfalls
*   **Coefficient Sharing:** Sharing mixture coefficients across all layers ($\alpha_m^{(i)} = \alpha_m$) is too restrictive for constructing INRs [S1].
*   **Independent Coefficients:** Setting mixture coefficients to different independent values across layers is more flexible but less stable due to more parameters and ignoring inter-layer dependency [S1]. The recommended approach is layer-specific coefficients projected from a latent vector [S1].
*   **Scalability:** The method exhibits limited scalability beyond fine-grained datasets. The latent space trained on diverse datasets (e.g., CIFAR-10) is not sufficiently smooth to be generalized by adopting a diffusion process [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class SIRENLayer:
    """SIREN layer with weighted mixture modulation."""
    def __init__(self, in_features, out_features, w0=30.0):
        self.w0 = w0
        self.in_features = in_features
        self.out_features = out_features

    def forward(self, x, weights, biases, alpha):
        """
        Compute weighted sum of basis network outputs, then apply sine activation.
        
        Args:
            x: Input tensor (batch_size, in_features)
            weights: List of M weight matrices, each (in_features, out_features)
            biases: List of M bias vectors, each (out_features,)
            alpha: Mixture coefficients for this layer, shape (M,)
            
        Returns:
            Output tensor (batch_size, out_features)
        """
        # Compute weighted sum of linear transforms
        # sum_{m=1}^M alpha_m * (W_m * x + b_m)
        weighted_sum = np.zeros((x.shape[0], self.out_features))
        for m in range(len(weights)):
            # Linear transform: W_m * x + b_m
            # Using matrix multiplication: (B, D_in) @ (D_in, D_out) -> (B, D_out)
            linear_out = x @ weights[m] + biases[m]
            weighted_sum += alpha[m] * linear_out
        
        # Apply sine activation with frequency scaling
        output = np.sin(self.w0 * weighted_sum)
        return output

class mNIF:
    """
    Mixture of Neural Implicit Functions.
    
    Args:
        M: Number of basis networks (mixtures).
        L: Number of hidden layers.
        W: Hidden layer width.
        d: Input dimension.
        k: Output dimension.
        H: Latent vector dimension.
    """
    def __init__(self, M, L, W, d, k, H):
        self.M = M
        self.L = L
        self.W = W
        self.d = d
        self.k = k
        self.H = H
        
        # Initialize M basis networks
        # Each basis has L+2 layers (input, L hidden, output)
        # We store weights and biases for each basis and each layer
        self.weights = [] # List of L+2 lists, each list has M matrices
        self.biases = []  # List of L+2 lists, each list has M vectors
        
        # Input layer: maps d -> W
        # Hidden layers: maps W -> W
        # Output layer: maps W -> k
        
        for i in range(L + 2):
            basis_weights = []
            basis_biases = []
            for m in range(M):
                if i == 0:
                    # Input layer
                    w = np.random.randn(d, W) * np.sqrt(1.0/d)
                    b = np.zeros(W)
                elif i == L + 1:
                    # Output layer
                    w = np.random.randn(W, k) * np.sqrt(1.0/W)
                    b = np.zeros(k)
                else:
                    # Hidden layer
                    w = np.random.randn(W, W) * np.sqrt(1.0/W)
                    b = np.zeros(W)
                basis_weights.append(w)
                basis_biases.append(b)
            self.weights.append(basis_weights)
            self.biases.append(basis_biases)
            
        # Projection matrix T: maps H -> M * (L+2) coefficients
        # T shape: (M * (L+2), H)
        self.T = np.random.randn(M * (L + 2), H) * np.sqrt(1.0/H)
        
        # w0 for SIREN
        self.w0 = 30.0

    def get_mixture_coefficients(self, phi):
        """
        Compute mixture coefficients for all layers from latent vector phi.
        
        Args:
            phi: Latent vector, shape (H,)
            
        Returns:
            List of Mixture coefficient arrays, one for each layer.
            Each array has shape (M,)
        """
        # alpha_all = T @ phi
        alpha_all = self.T @ phi
        # Reshape to (L+2, M)
        alpha_all = alpha_all.reshape(self.L + 2, self.M)
        return alpha_all

    def forward(self, x, phi):
        """
        Forward pass of the mNIF.
        
        Args:
            x: Query coordinates, shape (batch_size, d)
            phi: Latent vector, shape (H,)
            
        Returns:
            Output values, shape (batch_size, k)
        """
        # Get mixture coefficients for all layers
        alphas = self.get_mixture_coefficients(phi)
        
        # Input layer
        # h0 = x (no linear transform in standard INR definition for input layer? 
        # Source says h1 = f(0)(x). f(0) is input layer. 
        # Eq 2 says h(i+1) = sigma(W(i) h(i) + b(i)) for i in 1..L.
        # h1 = f(0)(x). Usually f(0) is just a linear layer or identity.
        # Source Eq 1: y = f(L+1) ... f(1) f(0)(x).
        # Source Eq 2: h(i+1) = ... for i in 1..L.
        # Source Eq 3: theta includes W(0)...W(L+1).
        # So f(0) is a linear layer.
        
        # Layer 0 (Input)
        # We need to handle the first layer separately if it doesn't have mixture coefficients in the same way,
        # or if it does. Eq 10 projects to alpha for all layers.
        # Let's assume all layers use the modulation.
        
        h = x
        for i in range(self.L + 2):
            # Get weights, biases, and alpha for layer i
            W_list = self.weights[i]
            b_list = self.biases[i]
            alpha = alphas[i]
            
            # Compute weighted sum
            weighted_sum = np.zeros((h.shape[0], self.W if i < self.L + 1 else self.k))
            for m in range(self.M):
                linear_out = h @ W_list[m] + b_list[m]
                weighted_sum += alpha[m] * linear_out
            
            # Apply activation
            if i < self.L + 1:
                # Hidden layers: Sine activation
                h = np.sin(self.w0 * weighted_sum)
            else:
                # Output layer: No activation specified in source for output, 
                # but SIREN usually uses sine for hidden. 
                # Source Eq 5: h(i+1) = sin(...) for i in 1..L.
                # Output layer f(L+1) is not explicitly defined with sine in Eq 5.
                # Standard INR Eq 2 uses sigma.
                # Let's assume linear output for regression/generation unless specified.
                # However, mNIF Eq 8 shows sin for the modulation step.
                # "mNIF layer operation: h(i+1) = sin(...)"
                # This implies all layers use sine?
                # Let's check Eq 8 context. "mNIF layer operation... This is the core modulation step".
                # It doesn't explicitly exclude the output layer.
                # But typically output is linear.
                # Given the ambiguity, and that SIREN uses sine for hidden, 
                # and Eq 5 defines SIREN for i in 1..L (hidden), 
                # we will apply sine to hidden layers and linear to output.
                h = weighted_sum
                
        return h

    def compute_loss(self, x, y_true, phi):
        """
        Compute reconstruction loss.
        """
        y_pred = self.forward(x, phi)
        loss = np.mean((y_pred - y_true) ** 2)
        return loss
```

## Subtopics

### Implicit Neural Networks (Neural Fields)
The parent method using implicit neural networks as continuous measurement representations. [Parent](16052-implicit-neural-networks-neural-fields.md)

### HyperDiffusion
A concurrent method that samples entire INR weights directly from a diffusion process, limiting it to voxel domains. mNIF uses latent vectors to modulate bases, allowing diverse modalities. [S1]

### GEM/GASP
Methods requiring access to training examples for manifold constraints (GEM) or using unstable permutation-invariant discriminators (GASP). mNIF avoids these costs and instabilities. [S1]

### Functa
Uses meta-learning with Hessian computation, which is computationally expensive. mNIF uses diffusion for latent sampling. [S1]

## Operating characteristics
*   **Time Complexity:** The computational complexity per input coordinate is small despite a large model capacity. The cost per query is invariant to the number of basis networks $M$ because the weighted sum is computed in parallel or efficiently [S1].
*   **Memory Complexity:** The model size for inference remains small compared to other methods due to the weighted model averaging strategy, which saves memory [S1]. The inference network size does not increase with $M$ [S1].
*   **Measured Performance:** On CelebA-HQ 64x64 with $(L,W,M,H)=(2,64,256,256)$, second-order meta-learning achieves PSNR 26.64 and rFID 24.37 [S1]. Auto-decoding with first-order gradients achieves PSNR 26.77 and rFID 24.37, surpassing meta-learning in performance and efficiency [S1].
*   **Scaling:** Performance saturates around $M=256$ [S1]. Increasing $H$ improves performance [S1].
*   **Strengths:** Efficient prediction, compact representation, versatile for diverse modalities (images, voxels, NeRF), easy to implement [S1].
*   **Limitations:** Limited scalability beyond fine-grained datasets; latent space may not be smooth enough for diverse datasets like CIFAR-10 when using diffusion [S1].

## Background
*   **Origin:** Proposed by Tackgeun You, Mijeong Kim, Jungtaek Kim, and Bohyung Han in the paper "Generative Neural Fields by Mixtures of Neural Implicit Functions" (arXiv:2310.19464) in October 2023 [S1].
*   **History:** The method builds on Implicit Neural Representations (INRs) and SIREN networks. It addresses limitations of HyperNet (instability) and FiLM (restrictiveness) by using weighted model averaging [S1].
*   **Usage:** Applicable to images, voxel data, and NeRF scenes. Used for sampling unseen data in specific target tasks, adopting denoising diffusion probabilistic models for sampling context vectors [S1].

## Sources
[S1] Generative Neural Fields by Mixtures of Neural Implicit Functions, https://doi.org/10.48550/arxiv.2310.19464, fetched 2026-10-29
