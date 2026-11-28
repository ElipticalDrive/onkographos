# Variational Compression of Implicit Neural Representations (VC-INR)

Also called: VC-INR, Modality-Agnostic Variational Compression of Implicit Neural Representations

## Overview
VC-INR is a modality-agnostic neural compression algorithm that compresses data by parameterizing it as an Implicit Neural Representation (INR) and applying non-linear transform coding to its compact latent representation [S1, unofficial]. It bridges the gap between latent coding and sparsity by using a soft gating mechanism to specialize a shared INR network for each data item, allowing the use of deep entropy coding without explicit quantization steps in the forward pass during inference [S1, unofficial]. Unlike conventional neural compression which relies on modality-specific analysis/synthesis transforms, VC-INR operates in a modality-agnostic space, outperforming prior INR-based methods and established codecs like MP3 and HEVC on various datasets [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A dataset of $N$ data points, where each point $x_i$ is represented as a set of coordinates $c \in \mathcal{C}$ and real-valued features $y \in \mathcal{Y}$ [S1, unofficial].
*   **Output:** A bit stream representing the compressed data, and a reconstructed signal $\hat{x}$ upon decoding [S1, unofficial].

**Data Structures:**
*   **Shared INR Weights ($\theta$):** A large matrix of weights shared across all data items, typically implemented as a SIREN MLP with 15 layers of 512 units [S1, unofficial].
*   **Latent Representation ($\phi$):** A compact, data-item-specific parameter vector. For VC-INR, $\phi$ is represented as a sequence of low-rank matrices $[U^{(1)}, V^{(1)}, \dots, U^{(L)}, V^{(L)}]$ for each layer $l$, predicted from a compact representation by reshaping a long vector [S1, unofficial].
*   **Gating Mask ($G_{low}^{(l)}$):** A soft gating matrix for layer $l$, defined as $G_{low}^{(l)} = \sigma(U^{(l)} (V^{(l)})^\top)$, where $\sigma$ is the sigmoid function, and $U^{(l)}, V^{(l)} \in \mathbb{R}^{m \times d}$ with $d \ll m$ [S1, unofficial].

**Parameters:**
*   $N$: Number of data points in the dataset [S1, unofficial].
*   $M$: Number of coordinate/feature pairs in a single data point [S1, unofficial].
*   $D$: Dimensionality of the INR representation weights [S1, unofficial].
*   $\omega_0$: Hyperparameter for the sinusoidal activation function [S1, unofficial].
*   $d$: Rank of the low-rank parameterization, where $d \ll m$ [S1, unofficial].
*   $\lambda$: Weight for the distortion term in the compression loss [S1, unofficial].
*   Sparsity threshold: Weights are considered sparse if their magnitude is less than 0.001 [S1, unofficial].

**Algorithm Steps:**

1.  **Meta-Learning Adaptation:** For a given data point $x$, obtain a compact latent representation $\phi^*$ by adapting the shared INR using Meta-Learning (e.g., MetaSGD) in a handful of optimization steps [S1, unofficial]. This avoids the high-dimensional parameter vectors of naive optimization [S1, unofficial].
2.  **Stabilization:** Apply LayerNorm to the modulation $\phi$ to stabilize gradients: $\phi \leftarrow \text{LayerNorm}(\phi)$ [S1, unofficial]. Use residual connections and increased layer widths in the predictor network [S1, unofficial].
3.  **Analysis Transform:** Encode the modulation $\phi$ into a code $z$ using an analysis transform network $g_a$:
    $$ z = g_a(\phi; \pi_a) $$
    where $\pi_a$ are the parameters of the analysis network [S1, unofficial].
4.  **Quantization:** Quantize the code $z$ to obtain discrete values $\hat{z}$:
    $$ \hat{z} = Q(z) = \text{round}(z) $$
    During training, approximate this with uniform noise $U(-\frac{1}{2}, \frac{1}{2})$ to generate a noisy code $\tilde{z}$ for differentiable optimization [S1, unofficial].
5.  **Entropy Coding:** Losslessly compress $\hat{z}$ using an entropy model (e.g., arithmetic or Huffman coding) to obtain a bit stream [S1, unofficial]. The entropy model $p_{\hat{z}}$ is a non-parametric piecewise linear density model [S1, unofficial].
6.  **Decoding:** Decode $\hat{z}$ to a reconstruction $\hat{\phi}$ using a synthesis transform network $g_s$:
    $$ \hat{\phi} = g_s(\hat{z}; \pi_s) $$
    where $\pi_s$ are the parameters of the synthesis network [S1, unofficial].
7.  **INR Reconstruction:** Use the reconstructed modulation $\hat{\phi}$ to specialize the shared INR. For each layer $l$, the transformation of activations $c^{(l-1)}$ is:
    $$ c^{(l-1)} \mapsto \sin(\omega_0 (G_{low}^{(l)} \odot W^{(l)} c^{(l-1)} + b^{(l)})) $$
    where $G_{low}^{(l)} = \sigma(U^{(l)} (V^{(l)})^\top)$ is derived from $\hat{\phi}$, $\odot$ is element-wise multiplication, and $W^{(l)}, b^{(l)}$ are the shared weights [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def layer_norm(x, eps=1e-5):
    """Standard Layer Normalization."""
    mean = np.mean(x, axis=-1, keepdims=True)
    var = np.var(x, axis=-1, keepdims=True)
    return (x - mean) / np.sqrt(var + eps)

def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-x))

def sinusoidal_activation(x, omega0):
    return np.sin(omega0 * x)

def vc_inr_compress(data_point_x, shared_inr_weights, analysis_net, synthesis_net, lambda_dist):
    """
    Compress a single data point using VC-INR.
    Note: This pseudocode represents the inference/encoding pipeline.
    Training involves optimizing pi_a and pi_s with the loss function below.
    """
    # Step 1: Meta-Learning Adaptation
    # In practice, this is done during training. 
    # For inference, phi* is assumed to be pre-computed or obtained via fast adaptation.
    phi_star = meta_learning_adaptation(data_point_x, shared_inr_weights)
    
    # Step 2: Stabilization
    phi_normalized = layer_norm(phi_star)
    
    # Step 3: Analysis Transform
    z = analysis_net(phi_normalized)
    
    # Step 4: Quantization
    z_hat = np.round(z)
    
    # Step 5: Entropy Coding (Abstracted)
    bit_stream = entropy_encode(z_hat)
    
    return bit_stream

def vc_inr_decompress(bit_stream, shared_inr_weights, synthesis_net, omega0):
    """
    Decompress a bit stream using VC-INR.
    """
    # Step 5 (Decode): Entropy Decoding
    z_hat = entropy_decode(bit_stream)
    
    # Step 6: Synthesis Transform
    phi_hat = synthesis_net(z_hat)
    
    # Step 7: INR Reconstruction
    # Reconstruct the gating masks from phi_hat
    # phi_hat contains flattened U and V matrices for each layer
    layers = []
    current_idx = 0
    for l in range(num_layers):
        # Reshape phi_hat to get U(l) and V(l)
        # Assuming phi_hat is structured as [U(1), V(1), ..., U(L), V(L)]
        U_l = phi_hat[:, current_idx : current_idx + m*d].reshape(-1, m, d)
        current_idx += m*d
        V_l = phi_hat[:, current_idx : current_idx + m*d].reshape(-1, m, d)
        current_idx += m*d
        
        # Compute Gating Mask
        G_low = sigmoid(U_l @ V_l.T) # Element-wise multiplication with W later
        
        layers.append((G_low, shared_inr_weights[l]))
        
    # Forward pass through INR with gating
    # This is a simplified representation; actual implementation requires 
    # handling of coordinate inputs and layer-by-layer computation.
    def reconstruct_signal(coordinates):
        c = coordinates
        for l in range(num_layers):
            G_low, W_l = layers[l]
            b_l = shared_inr_weights[l].bias
            # Apply gating and activation
            # Note: In the paper, G_low is applied element-wise to W_l * c
            # Assuming W_l is (m, m) and G_low is (m, m)
            pre_act = W_l @ c + b_l
            gated_act = G_low * pre_act # Element-wise multiplication
            c = sinusoidal_activation(gated_act, omega0)
        return c

    return reconstruct_signal

def compression_loss(phi, analysis_net, synthesis_net, shared_inr_weights, data_point_x, lambda_dist):
    """
    Compute the compression loss for training.
    """
    # Analysis
    z = analysis_net(phi)
    
    # Quantization approximation (uniform noise)
    z_tilde = z + np.random.uniform(-0.5, 0.5, size=z.shape)
    
    # Entropy rate (using the density model p_z_tilde)
    # In practice, this is computed using the piecewise linear density model
    rate = -np.log2(entropy_model_density(z_tilde))
    
    # Synthesis
    phi_hat = synthesis_net(np.round(z))
    
    # Distortion: MSE between reconstructed phi and original phi
    # Or directly on the data signal if L_distortion is modified
    distortion = np.mean((phi_hat - phi) ** 2)
    
    # Total Loss
    loss = rate + lambda_dist * distortion
    return loss
```

**Edge Cases and Pitfalls:**
*   **Instability:** Direct optimization of non-linear networks via Meta-Learning is unstable. This is mitigated by LayerNorm on $\phi$, residual connections, and increasing layer widths [S1, unofficial].
*   **Large Images:** For large images (e.g., Kodak dataset), images are divided into smaller patches to handle input size [S1, unofficial].
*   **Audio/Video:** Patching is used for audio (LibriSpeech) and video (UCF-101) to keep comparisons fair with baselines like COIN++ [S1, unofficial].
*   **Sparsity Definition:** Sparsity is defined as $|(G_{low} \odot W)_{ij}| < 0.001$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Meta-Learning allows $\phi^*$ to be obtained in a handful of optimization steps, reducing encoding time compared to naive per-signal optimization [S1, unofficial].
*   **Memory Complexity:** The shared INR weights $\theta$ are stored once, while only the compact latent representation $\phi$ (low-rank matrices) is stored per data item, significantly reducing memory usage compared to storing full INR weights for each item [S1, unofficial].
*   **Performance:**
    *   CIFAR-10: Improves PSNR by 3.3 dB at the same bpp rate compared to previous INR-based methods [S1, unofficial].
    *   Kodak: Improves PSNR by 2 dB at the same bpp rate [S1, unofficial].
    *   ERA5 Climate Data: Improves PSNR by 3.5 dB at the same bpp rate [S1, unofficial].
    *   LibriSpeech Audio: Improves PSNR by 9.5 dB at the same bpp rate, outperforming MP3 by 5.6 dB [S1, unofficial].
    *   Video (UCF-101): Outperforms HEVC by 8.8 dB [S1, unofficial].
*   **Strengths:** Modality-agnostic, high reconstruction quality at low bitrates, avoids approximate inference required by sparse methods, leverages deep entropy coding [S1, unofficial].
*   **Limitations:** Does not take advantage of modality-specific inductive biases, which may limit performance compared to hand-designed codecs in some cases [S1, unofficial].

## Background
VC-INR was introduced by Jonathan Richard Schwarz et al. from DeepMind, University College London, KAIST, and POSTECH in 2023 at the 40th International Conference on Machine Learning (ICML) [S1, unofficial]. It builds on the work of Ballé et al. (2017) on transform coding and Dupont et al. (2022) on INR compression [S1, unofficial]. The method addresses the lack of generally accepted compression techniques for novel data modalities such as AR/VR, point clouds, remote sensing, and climate data [S1, unofficial].

## Sources
[S1] Modality-Agnostic Variational Compression of Implicit Neural Representations, https://proceedings.mlr.press/v202/schwarz23a.html, fetched 2026-10-29 (unofficial)
