# Codebook Model (VQ-VAE)
Also called: VQ-VAE, Vector Quantized Variational Autoencoder

## Overview
The Codebook Model (VQ-VAE) maps continuous input data, such as images, to a smaller discrete latent space (code space) and reconstructs them back to the original domain [S1]. It differs from standard Variational Autoencoders by using a categorical distribution over the latent space, replacing continuous latent variables with discrete tokens selected from a learned codebook [S1]. This discretization enables efficient sequence modeling (e.g., by Transformers) and reduces dimensionality significantly, as demonstrated by mapping 128x128 pixel images to 8x8 tokens [S1].

## How it works
**Inputs and Outputs:**
*   Input: Image $x$ of spatial dimensions $H \times W$ (e.g., $128 \times 128$).
*   Output: Reconstructed image $\hat{x}$ of the same dimensions.
*   Intermediate: Discrete latent tokens (indices) of spatial dimensions $h \times w$ (e.g., $8 \times 8$).

**Data Structures:**
*   **Encoder Convolutional Part ($f_\theta^{(enc)}$):** A stack of strided convolutional layers that reduces spatial resolution. It outputs a feature map $z \in \mathbb{R}^{h \times w \times d_{lat}}$.
*   **Codebook ($W^{(emb)}$):** A matrix of embedding vectors $W^{(emb)} \in \mathbb{R}^{n_{lat} \times d_{lat}}$, where $n_{lat}$ is the number of codebook entries and $d_{lat}$ is the latent dimension. Each row $W_k^{(emb)}$ is a vector of length $d_{lat}$.
*   **Decoder Convolutional Part ($D_\theta$):** A stack of convolutional layers combined with upscaling operations that increase spatial resolution from $h \times w$ back to $H \times W$.

**Numbered Steps:**
1.  **Encoding:** The encoder $E_\theta$ processes the input image $x$ through strided convolutional layers to produce a continuous feature map $z = f_\theta^{(enc)}(x)$ with spatial dimensions $h \times w$ and channel depth $d_{lat}$ [S1].
2.  **Quantization:** For each spatial position $(i, j)$ in the feature map $z$, find the index $k$ of the closest embedding vector in the codebook. The discrete latent representation $z_q$ is obtained by replacing $z_{i,j}$ with the corresponding codebook vector $W_k^{(emb)}$ [S1].
    The index selection is defined as:
    $$ k = \arg\min_k \| z_{i,j} - W_k^{(emb)} \|_2 $$
    where $z_{i,j} = (f_\theta^{(enc)}(x))_{i,j}$ [S1].
3.  **Gradient Approximation:** Since the $\arg\min$ operation is non-differentiable, gradients are propagated from the decoder to the encoder using a straight-through estimator. This copies the gradients from the input of the decoder (the quantized vectors) directly to the output of the encoder (the continuous vectors) [S1].
4.  **Decoding:** The decoder $D_\theta$ takes the quantized feature map $z_q$ (indices mapped back to vectors via $W^{(emb)}$) and applies convolutional layers with upscaling to reconstruct the image $\hat{x}$ at the original resolution [S1].
5.  **Codebook Update:** The embedding vectors in the codebook are updated using exponential moving averages (EMA) rather than standard backpropagation alone, to stabilize training [S1].

**Loss Function:**
The total loss $\mathcal{L}$ is a weighted sum of three components [S1]:
1.  **Reconstruction Loss:** Pixel-wise Mean Absolute Error (MAE) between input $x$ and reconstruction $\hat{x}$.
2.  **Perceptual Loss:** Feature-space distance between $x$ and $\hat{x}$ [S1].
3.  **Commitment Loss ($\mathcal{L}_c$):** Encourages the encoder outputs to stay close to the chosen embedding vectors to prevent frequent switching between codes [S1]:
    $$ \mathcal{L}_c = \min_k \| z_{i,j} - \text{sg}(W_k^{(emb)}) \|_2^2 $$
    where $\text{sg}(\cdot)$ denotes the stop-gradient operation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VQVAECodebook:
    def __init__(self, n_lat, d_lat, ema_decay=0.99):
        """
        Initialize the VQ-VAE codebook.
        
        Args:
            n_lat (int): Number of embedding vectors in the codebook.
            d_lat (int): Dimensionality of each embedding vector.
            ema_decay (float): Decay rate for exponential moving average updates.
        """
        self.n_lat = n_lat
        self.d_lat = d_lat
        self.ema_decay = ema_decay
        
        # Initialize codebook embeddings
        # W_emb shape: (n_lat, d_lat)
        self.W_emb = nn.Parameter(torch.randn(n_lat, d_lat))
        
        # EMA statistics for codebook update
        self.ema_cluster_size = torch.zeros(n_lat)
        self.ema_w_sum = torch.zeros_like(self.W_emb)

    def encode(self, x, encoder_conv_layers):
        """
        Encode input image to continuous latent space.
        
        Args:
            x (Tensor): Input image of shape (B, C, H, W).
            encoder_conv_layers (ModuleList): Strided convolutional layers.
            
        Returns:
            z (Tensor): Continuous latent feature map of shape (B, h, w, d_lat).
        """
        # Apply strided convolutions to reduce spatial dimensions
        z = encoder_conv_layers(x)
        # Reshape to (B * h * w, d_lat) for quantization
        B, C, H, W = z.shape
        z_flat = z.permute(0, 2, 3, 1).contiguous().view(-1, C)
        return z_flat, z.shape

    def quantize(self, z_flat):
        """
        Map continuous latent vectors to discrete codebook indices.
        
        Args:
            z_flat (Tensor): Flattened latent vectors of shape (N, d_lat).
            
        Returns:
            indices (Tensor): Indices of closest codebook entries of shape (N,).
            z_q_flat (Tensor): Quantized vectors of shape (N, d_lat).
        """
        # Calculate distances between each latent vector and all codebook entries
        # dists shape: (N, n_lat)
        dists = torch.cdist(z_flat, self.W_emb, p=2)
        
        # Find index of minimum distance
        indices = torch.argmin(dists, dim=1)
        
        # Gather the corresponding embedding vectors
        z_q_flat = self.W_emb[indices]
        
        return indices, z_q_flat

    def straight_through_gradient(self, z_flat, z_q_flat):
        """
        Approximate gradient for the non-differentiable quantization step.
        
        Args:
            z_flat (Tensor): Original continuous vectors.
            z_q_flat (Tensor): Quantized vectors.
            
        Returns:
            z_q_st (Tensor): Quantized vectors with straight-through gradient.
        """
        # Stop gradient on z_q_flat for the forward pass, 
        # but pass gradients from z_q_flat to z_flat in backward pass
        z_q_st = z_flat + (z_q_flat - z_flat).detach()
        return z_q_st

    def update_codebook_ema(self, z_flat, indices):
        """
        Update codebook embeddings using Exponential Moving Average.
        
        Args:
            z_flat (Tensor): Continuous latent vectors of shape (N, d_lat).
            indices (Tensor): Indices of selected codebook entries of shape (N,).
        """
        # Count occurrences of each codebook entry
        cluster_size = torch.bincount(indices, minlength=self.n_lat).float()
        
        # Sum of vectors assigned to each codebook entry
        w_sum = torch.zeros_like(self.W_emb)
        for i in range(self.n_lat):
            mask = (indices == i)
            w_sum[i] = z_flat[mask].sum(dim=0)
            
        # Update EMA statistics
        self.ema_cluster_size = self.ema_decay * self.ema_cluster_size + (1 - self.ema_decay) * cluster_size
        self.ema_w_sum = self.ema_decay * self.ema_w_sum + (1 - self.ema_decay) * w_sum
        
        # Handle dead codes (cluster_size == 0)
        # Not given in the sources for specific handling, but standard practice involves 
        # reinitializing dead codes. We assume sources imply standard EMA update [S1].
        
        # Normalize and update weights
        n = self.ema_cluster_size.sum()
        normalized_size = self.ema_cluster_size / (n + 1e-5)
        normalized_w_sum = self.ema_w_sum / (n + 1e-5)
        
        self.W_emb.data = normalized_w_sum

    def compute_commitment_loss(self, z_flat, indices):
        """
        Compute the commitment loss.
        
        Args:
            z_flat (Tensor): Continuous latent vectors of shape (N, d_lat).
            indices (Tensor): Indices of selected codebook entries of shape (N,).
            
        Returns:
            loss (Tensor): Scalar commitment loss.
        """
        # Get the embedding vectors corresponding to the indices
        w_k = self.W_emb[indices]
        
        # Commitment loss: distance between encoder output and codebook vectors
        # Using stop-gradient on codebook vectors
        loss = torch.mean((z_flat - w_k.detach()) ** 2)
        return loss

    def forward(self, x, encoder_conv_layers, decoder_conv_layers):
        """
        Full forward pass of the Codebook Model.
        
        Args:
            x (Tensor): Input image.
            encoder_conv_layers (ModuleList): Encoder convolutional layers.
            decoder_conv_layers (ModuleList): Decoder convolutional layers.
            
        Returns:
            x_recon (Tensor): Reconstructed image.
            loss (Tensor): Total loss.
        """
        # 1. Encode
        z_flat, orig_shape = self.encode(x, encoder_conv_layers)
        
        # 2. Quantize
        indices, z_q_flat = self.quantize(z_flat)
        
        # 3. Straight-through gradient
        z_q_st = self.straight_through_gradient(z_flat, z_q_flat)
        
        # 4. Decode
        # Reshape z_q_st back to spatial dimensions for decoder
        B, H, W, C = orig_shape
        z_q_reshaped = z_q_st.view(B, H, W, C).permute(0, 3, 1, 2)
        x_recon = decoder_conv_layers(z_q_reshaped)
        
        # 5. Compute Losses
        # Reconstruction Loss (MAE)
        recon_loss = torch.mean(torch.abs(x - x_recon))
        
        # Perceptual Loss (Placeholder for specific implementation)
        # Not given in the sources for exact formula, assumed to be a standard perceptual loss [S1]
        perceptual_loss = 0.0 # Placeholder
        
        # Commitment Loss
        commitment_loss = self.compute_commitment_loss(z_flat, indices)
        
        # Total Loss (Weighted sum)
        # Weights not given in the sources [S1]
        loss = recon_loss + perceptual_loss + commitment_loss
        
        # 6. Update Codebook (EMA)
        self.update_codebook_ema(z_flat, indices)
        
        return x_recon, loss
```

## Operating characteristics
*   **Time Complexity:** Dominated by the encoder and decoder convolutional operations and the distance calculation in quantization. Quantization requires computing distances between $N$ latent vectors and $n_{lat}$ codebook entries, costing $O(N \cdot n_{lat} \cdot d_{lat})$.
*   **Memory Complexity:** Requires storing the codebook matrix $W^{(emb)}$ of size $n_{lat} \times d_{lat}$ and EMA statistics. The latent representation is significantly smaller than the input (e.g., $8 \times 8$ tokens vs $128 \times 128$ pixels) [S1].
*   **Strengths:** Discrete latent space enables the use of autoregressive models (like Transformers) for generation and compression. It reduces dimensionality effectively [S1].
*   **Limitations:** The quantization step is non-differentiable, requiring the straight-through estimator which can lead to training instability if not managed (e.g., via EMA updates) [S1].
*   **When to use:** When a discrete, compressed representation of visual data is needed for sequence modeling or efficient storage, particularly in view synthesis tasks [S1].

## Background
The Codebook Model (VQ-VAE) is based on the Vector Quantized Variational Autoencoder architecture [S1]. It was introduced to provide a categorical distribution over the latent space, contrasting with the continuous Gaussian distributions of standard VAEs [S1]. In the context of neural rendering, it is used to map images to a discrete code space, allowing Transformers to operate on a simplified, tokenized representation of views [S1]. The method relies on exponential moving average updates for the codebook to stabilize training [S1].

## Sources
[S1] ViewFormer: NeRF-Free Neural Rendering from Few Images Using Transformers, https://doi.org/10.1007/978-3-031-19784-0_12, fetched 2026-10-29
