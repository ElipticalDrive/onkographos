# 3D Shape Autoencoder

Also called: Shape prior autoencoder

## Overview
The 3D Shape Autoencoder is a convolutional neural network trained to learn a prior of 3D shapes using 3D convolutions [S1, unofficial]. It maps high-dimensional 3D inputs into a low-dimensional latent vector and reconstructs them, enabling tasks such as novel-view synthesis and image relighting [S1, unofficial]. It replaces feed-forward networks with a 3D convolutional architecture to explicitly learn a shape prior, allowing the recovery of pose and shape factors from single images [S1, unofficial].

## How it works
The method consists of an encoder, a bottleneck, and a decoder, all utilizing 3D convolutions to process volumetric data [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A 3D volume or shape representation (e.g., a voxel grid or signed distance function) [S1, unofficial].
*   **Output:** A reconstructed 3D volume identical in dimensions to the input [S1, unofficial].

**Data Structures and Layout:**
The network parameters are defined by specific channel counts, kernel sizes, and strides [S1, unofficial]. The activation functions are strictly defined per layer type [S1, unofficial].

1.  **Encoder:**
    *   Consists of a series of 3D convolutional layers [S1, unofficial].
    *   **Layer 1:** Channels = 64, Kernel Size = 5, Stride = 2 [S1, unofficial].
    *   **Layer 2:** Channels = 128, Kernel Size = 5, Stride = 2 [S1, unofficial].
    *   **Layer 3:** Channels = 256, Kernel Size = 2, Stride = 2 [S1, unofficial].
    *   **Layer 4:** Channels = 512, Kernel Size = 2, Stride = 2 [S1, unofficial].
    *   **Activation:** Exponential Linear Unit (ELU) is applied after every convolutional layer in the encoder, except for the final one [S1, unofficial].

2.  **Bottleneck (Middle):**
    *   A fully connected (linear) layer maps the flattened output of the last encoder convolution to a 200-dimensional vector [S1, unofficial].
    *   **Activation:** Sigmoid activation is applied to this 200-dimensional vector [S1, unofficial].
    *   A second fully connected layer maps the 200-dimensional vector to a $(4 \times 4 \times 512)$-dimensional vector (total 8192 elements) [S1, unofficial].

3.  **Decoder:**
    *   The vector from the bottleneck is reshaped into a tensor of size $4 \times 4 \times 512$ [S1, unofficial].
    *   Consists of a series of 3D transposed convolutional (up-convolution) layers [S1, unofficial].
    *   **Layer 1:** Channels = 256, Kernel Size = 4, Stride = 2 [S1, unofficial].
    *   **Layer 2:** Channels = 128, Kernel Size = 4, Stride = 2 [S1, unofficial].
    *   **Layer 3:** Channels = 64, Kernel Size = 4, Stride = 2 [S1, unofficial].
    *   **Layer 4:** Channels = 32, Kernel Size = 4, Stride = 2 [S1, unofficial].
    *   **Layer 5:** Channels = 1, Kernel Size = 4, Stride = 1 [S1, unofficial].
    *   **Activation:** ELU is applied after every convolutional layer in the decoder, except for the final one [S1, unofficial].

**Formulas and Parameters:**
*   **Activation Functions:**
    *   Hidden layers: $ELU(x) = \max(0, x) + \min(0, \alpha(e^x - 1))$ where $\alpha=1.0$ is the standard default for ELU [S1, unofficial].
    *   Final layers (Encoder output, Decoder output): $Sigmoid(x) = \frac{1}{1 + e^{-x}}$ [S1, unofficial].
*   **Reshaping:** The 200-dim vector $v$ is reshaped to tensor $T$ such that $T[i, j, k]$ corresponds to the appropriate index in $v$ based on row-major ordering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward(input_volume):
    # Encoder
    x = elu(conv3d(input_volume, in_channels=1, out_channels=64, kernel_size=5, stride=2))
    x = elu(conv3d(x, in_channels=64, out_channels=128, kernel_size=5, stride=2))
    x = elu(conv3d(x, in_channels=128, out_channels=256, kernel_size=2, stride=2))
    x = sigmoid(conv3d(x, in_channels=256, out_channels=512, kernel_size=2, stride=2))
    
    # Bottleneck
    x = flatten(x)
    x = linear(x, 200)
    x = sigmoid(x)
    x = linear(x, 4 * 4 * 512)
    x = reshape(x, (4, 4, 512))
    
    # Decoder
    x = elu(conv_transpose3d(x, in_channels=512, out_channels=256, kernel_size=4, stride=2))
    x = elu(conv_transpose3d(x, in_channels=256, out_channels=128, kernel_size=4, stride=2))
    x = elu(conv_transpose3d(x, in_channels=128, out_channels=64, kernel_size=4, stride=2))
    x = elu(conv_transpose3d(x, in_channels=64, out_channels=32, kernel_size=4, stride=2))
    x = sigmoid(conv_transpose3d(x, in_channels=32, out_channels=1, kernel_size=4, stride=1))
    
    return x
```

**Edge Cases and Pitfalls:**
*   **Input Dimensions:** The architecture assumes an input size that reduces to $4 \times 4$ spatially after the encoder's strided convolutions. If the input resolution is not compatible with the kernel/stride sequence (5,5,2,2 with strides 2,2,2,2), the final spatial dimensions before the bottleneck may not be $4 \times 4$, causing the reshape operation to fail or produce incorrect latent tensors [S1, unofficial].
*   **Activation Mismatch:** Using ReLU or other activations for the specified layers will deviate from the trained prior, potentially leading to poor reconstruction quality or mode collapse in generative tasks [S1, unofficial].
*   **Padding:** Padding values for the convolutional layers are not specified in the sources [G].

## Subtopics

### [Conditional Adversarial Autoencoder](14796-conditional-adversarial-autoencoder.md)
The Conditional Adversarial Autoencoder (CAE) is a neural network compression method that adapts a pre-trained conditional Generative Adversarial Network (GAN) into a reconstruction pipeline by removing the explicit latent code injection step. It solves the problem of generating high-fidelity 3D volumes from single images by leveraging the GAN's learned prior for adversarial regularization while relying on an autoencoder structure for direct reconstruction. Unlike standard autoencoders that optimize only reconstruction loss, or full GANs that require latent sampling, the CAE trades off the flexibility of latent space exploration for a more direct, albeit potentially inconsistent, optimization path. It is distinct from the Latent Voxel Auto-Encoder, which explicitly maintains and optimizes a discrete latent voxel grid for storage efficiency [17589-latent-voxel-auto-encoder.md].

### [Latent Voxel Auto-Encoder](17589-latent-voxel-auto-encoder.md)
The Latent Voxel Auto-Encoder (LAE) is a hierarchical neural network that compresses 3D voxel grids into a structured latent space consisting of a global 1D vector, a coarse 3D tensor, and a fine 2D tensor. It solves the dimensionality bottleneck of training diffusion models on high-resolution 3D data by reducing the representation size while preserving semantic structure. Unlike standard auto-encoders that map directly to a single latent space, LAE separates global scene properties from local geometric details using 2D CNNs applied to concatenated vertical slices, enabling efficient generation in [Hierarchical Latent Diffusion Models](17598-hierarchical-latent-diffusion-models.md).

### SurroFlow
SurroFlow uses normalizing flows to learn an invertible transformation between a simple base distribution and the data distribution [S2, unofficial]. It trains a conditional normalizing flow in the latent space of a 3D convolutional autoencoder [S2, unofficial]. It supports bidirectional prediction: forward mapping from parameters to simulation outputs, and reverse mapping from observed data back to likely parameter values [S2, unofficial]. The constant-dimensionality constraint of normalizing flows limits SurroFlow to operating at lower resolution than the native simulation grid [S2, unofficial]. On MPAS-Ocean (192 × 96 × 12, 128 training members), it achieved 46.69 dB PSNR with a 48 MB model trained in 11.5 hours [S2, unofficial]. On Nyx at $128^3$ rather than $512^3$, PSNR drops to 30.93 dB, below VDL-Surrogate’s 33.66 dB [S2, unofficial]. Compared to VDL-Surrogate’s 45.82 dB at 0.63 GB and 139.8 hours, SurroFlow offers significantly faster training and smaller model size [S2, unofficial].

### 3Da-AE Training
This variant involves a specific training procedure for 3Da-AE [S3, unofficial].
**Steps:**
1.  **Input:** Training set $X_{train}$, Encoder $E_\phi$, Decoder $D_\psi$, Renderer $R_\alpha$, number of steps $N$, loss weights $\lambda_{ae}, \lambda_{Enc}, \lambda_{Dec}$, and optimizer [S3, unofficial].
2.  **Initialization:** Random initialization of local triplanes $T_{mic}$, weights $W$, and global triplanes $T_{mac}$ [S3, unofficial].
3.  **Loop:** For $N$ steps, for each batch $\{s, p, x_p\}$ in shuffled $X_{train}$:
    *   Compute local representation $T_{smic}$ and global representation $T_{smac}$ from $T_{mic}, W_s, B$ [S3, unofficial].
    *   Compute total representation $T_s = T_{smic} \oplus T_{smac}$, where $\oplus$ concatenates along the feature dimension [S3, unofficial].
    *   Encode input: $z_p = E_\phi(x_p)$ [S3, unofficial].
    *   Decode latent: $\hat{x}_p = D_\psi(z_p)$ [S3, unofficial].
    *   Render from triplanes: $\tilde{z}_p = R_\alpha(T_s, p)$ [S3, unofficial].
    *   Decode rendered latent: $\tilde{x}_p = D_\psi(\tilde{z}_p)$ [S3, unofficial].
    *   Compute losses:
        *   $L_{ae} = \|x_p - \hat{x}_p\|^2$ [S3, unofficial].
        *   $L_{Enc} = \|z_p - \tilde{z}_p\|^2$ [S3, unofficial].
        *   $L_{Dec} = \|x_p - \tilde{x}_p\|^2$ [S3, unofficial].
    *   Compute total loss: $L = \lambda_{ae}L_{ae} + \lambda_{Enc}L_{Enc} + \lambda_{Dec}L_{Dec}$ [S3, unofficial].
    *   Optimize $T_{mic}, W_s, B, \alpha, \phi, \psi$ with $L$ [S3, unofficial].
**Pitfall:** During warmup, only local triplanes, global triplanes, and scene-specific coefficients are optimized; the autoencoder is kept frozen to prevent random gradients from back-propagating into $E_\phi$ and $D_\psi$ [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources for the base 3D Shape Autoencoder. Variants report specific metrics, e.g., SurroFlow achieves 46.69 dB PSNR on MPAS-Ocean with a 48 MB model [S2, unofficial].
*   **Strengths:** Effective at learning shape priors for tasks like novel-view synthesis and relighting [S1, unofficial]. The use of 3D convolutions preserves spatial structure better than feed-forward networks [S1, unofficial].
*   **Limitations:** Speed and performance require further improvement [S1, unofficial].
*   **When to use:** When a compact prior of 3D shapes is needed for generative tasks or reconstruction from limited data [S1, unofficial].
*   **When not to use:** When high-resolution simulation grids are required without resolution reduction, as normalizing flow variants are limited by constant-dimensionality constraints [S2, unofficial].

## Background
The 3D Shape Autoencoder was introduced in the context of differentiable rendering and 3D shape representation [S1, unofficial]. It was proposed as an alternative to feed-forward networks, aiming to learn a prior of 3D shapes using 3D convolutions [S1, unofficial]. The method was used to reconstruct chairs from single images, recovering pose and shape factors for sharper novel-view synthesis and relighting [S1, unofficial]. It is a foundational component in methods that require a structured latent space for 3D data, such as the SurroFlow variant for scientific simulations [S2, unofficial] and the 3Da-AE for efficient scene learning [S3, unofficial].

## Sources
[S1] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S3] Exploring 3D-aware Latent Spaces for Efficiently Learning Numerous Scenes, https://doi.org/10.48550/arXiv.2403.11678, fetched 2026-10-29 (unofficial)
