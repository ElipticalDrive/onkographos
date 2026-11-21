# SF-VAE (Spherical Folding Variational Autoencoder)
Also called: SFVAE, Spherical Folding VAE

## Overview
SF-VAE is a learning-based initialization method for 3D Gaussian Splatting that uses a Variational Autoencoder (VAE) to map input point cloud data into a latent space and reconstructs Gaussian parameters via a spherical folding grid [S1, unofficial]. It differs from heuristic initialization methods by enforcing geometric consistency through a learned latent representation, making it suitable for scenes where data-driven priors improve convergence stability [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A batch of point cloud data. The source code indicates an input dimensionality of 7 per sample, interpreted as 3D centroid coordinates $(x, y, z)$ and potentially normal components or initial feature embeddings [S1, unofficial].
*   **Output:** Reconstructed 3D Gaussian parameters (centroids, scales, opacities) and associated color features (Spherical Harmonics coefficients) [S1, unofficial].

**Data Structures:**
*   **SFModel:** The core neural network module.
    *   `input_dim`: 7 [S1, unofficial].
    *   `embedding_dim`: Latent vector $z$ dimension. Default: 32 [S1, unofficial].
    *   `grid_dim`: Dimension of the 2D spherical folding grid. Default: 24 [S1, unofficial].
*   **SFDecoder Grid:** A 2D grid of shape `[grid_dim^2, 2]` representing local coordinates on the sphere, repeated for the batch size [S1, unofficial].
*   **Gaussian Parameters:** The output is structured into:
    *   Centroids ($xyz$)
    *   Scales ($scale$)
    *   Rotations ($rotation$)
    *   Opacity ($opacity$)
    *   Color features ($feat\_dc$, $feat\_extra$) [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Instantiate `SFModel` with specified `embedding_dim` (default 32), `grid_dim` (default 24), and VAE mode flags [S1, unofficial]. Initialize the Chamfer Loss criterion with `geo_weight=0.5` [S1, unofficial].
2.  **Encoding:** Pass the input point cloud through the encoder network to produce the mean ($\mu$) and log-variance ($\log\sigma^2$) of the latent distribution [S1, unofficial].
3.  **Reparameterization:** Sample the latent vector $z$ from the distribution $\mathcal{N}(\mu, \sigma^2)$ using the reparameterization trick [S1, unofficial].
4.  **Decoding (Spherical Folding):**
    *   The decoder utilizes a spherical folding grid of size `grid_dim` x `grid_dim` [S1, unofficial].
    *   The latent vector $z$ is processed to predict offsets or parameters relative to this grid structure, effectively "folding" the 2D grid representation into 3D Gaussian properties [S1, unofficial].
    *   The specific mapping from the latent code to the Gaussian parameters (centroids, scales, rotations, opacities, and SH coefficients) is performed by the decoder network [S1, unofficial].
5.  **Loss Calculation:**
    *   **Chamfer Loss:** Compute the Chamfer distance between the input points and the reconstructed centroids. The loss is weighted by `geo_weight` (default 0.5) [S1, unofficial].
    *   **KL Divergence:** If operating in VAE mode, compute the Kullback-Leibler divergence loss $L_{KL}$ to regularize the latent space:
        $$ L_{KL} = -0.5 \times \frac{1}{N} \sum_{i=1}^{N} (1 + \log\sigma_i^2 - \mu_i^2 - \sigma_i^2) $$
        where $N$ is the batch size, and $\mu_i, \sigma_i^2$ are the mean and variance for the $i$-th sample in the batch [S1, unofficial].
    *   **Normalization Loss:** An optional normalization loss term can be applied, weighted by `norm_weight` (default 0) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SFVAE(nn.Module):
    def __init__(self, embedding_dim=32, grid_dim=24, norm_weight=0.0):
        super(SFVAE, self).__init__()
        self.embedding_dim = embedding_dim
        self.grid_dim = grid_dim
        self.norm_weight = norm_weight
        
        # Encoder: Maps input (dim 7) to latent mean and log-variance
        self.encoder = nn.Sequential(
            nn.Linear(7, 128),
            nn.ReLU(),
            nn.Linear(128, 128),
            nn.ReLU(),
            nn.Linear(128, embedding_dim * 2) # Output: [mu, logvar]
        )
        
        # Decoder: Maps latent vector to Gaussian parameters
        # Note: Exact architecture details (layers) are not fully specified in source
        # but it outputs the necessary Gaussian attributes [S1, unofficial]
        self.decoder = nn.Sequential(
            nn.Linear(embedding_dim, 128),
            nn.ReLU(),
            nn.Linear(128, 128),
            nn.ReLU(),
            nn.Linear(128, 7) # Placeholder for output dimensionality logic
        )
        
        # Spherical Folding Grid
        # Creates a grid of shape [grid_dim^2, 2] representing local coordinates
        self.grid = self._create_spherical_grid(grid_dim)
        
        # Loss criterion
        self.criterion = ChamferLoss(geo_weight=0.5)

    def _create_spherical_grid(self, grid_dim):
        # Generate grid points on a sphere or 2D manifold
        # Shape: [grid_dim^2, 2]
        # Implementation detail: [G] Standard grid generation logic
        u = torch.linspace(0, 1, grid_dim)
        v = torch.linspace(0, 1, grid_dim)
        U, V = torch.meshgrid(u, v)
        grid = torch.stack([U.flatten(), V.flatten()], dim=-1)
        return grid

    def reparameterize(self, mu, logvar):
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        return mu + eps * std

    def forward(self, x):
        # x shape: [Batch, 7]
        # 1. Encode
        params = self.encoder(x)
        mu = params[:, :self.embedding_dim]
        logvar = params[:, self.embedding_dim:]
        
        # 2. Reparameterize
        z = self.reparameterize(mu, logvar)
        
        # 3. Decode
        # The decoder uses the latent z and potentially the grid structure
        # to reconstruct Gaussian parameters [S1, unofficial]
        reconstructed = self.decoder(z)
        
        # 4. Compute Losses
        # Chamfer Loss between input x (centroids) and reconstructed centroids
        # Note: x[:, :3] are the centroids
        loss_chamfer = self.criterion(x[:, :3], reconstructed[:, :3])
        
        # KL Divergence Loss
        loss_kl = -0.5 * torch.mean(1 + logvar - mu.pow(2) - logvar.exp())
        
        # Normalization Loss (if enabled)
        loss_norm = torch.tensor(0.0)
        if self.norm_weight > 0:
            # [G] Placeholder for specific normalization logic
            pass
            
        total_loss = loss_chamfer + loss_kl + self.norm_weight * loss_norm
        
        return reconstructed, total_loss, loss_chamfer, loss_kl

class ChamferLoss(nn.Module):
    def __init__(self, geo_weight=0.5):
        super(ChamferLoss, self).__init__()
        self.geo_weight = geo_weight

    def forward(self, x, y):
        # Compute Chamfer distance between two point sets
        # x: [Batch, N, 3], y: [Batch, M, 3]
        # [G] Standard Chamfer distance implementation using pairwise distances
        # This is a simplified representation; actual implementation requires
        # efficient nearest neighbor search (e.g., using torch.cdist)
        dist_x = torch.cdist(x, y)
        dist_y = torch.cdist(y, x)
        
        min_dist_x, _ = torch.min(dist_x, dim=2)
        min_dist_y, _ = torch.min(dist_y, dim=2)
        
        loss = torch.mean(min_dist_x) + torch.mean(min_dist_y)
        return loss * self.geo_weight
```

**Parameters:**
*   `embedding_dim`: Dimensionality of the latent vector $z$. Default: 32 [S1, unofficial].
*   `grid_dim`: Dimension of the spherical folding grid. Default: 24 [S1, unofficial].
*   `norm_weight`: Weight for the normalization loss term. Default: 0 [S1, unofficial].
*   `geo_weight`: Weight for the Chamfer loss term. Default: 0.5 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The use of floating-point arithmetic for the spherical grid and latent sampling requires careful handling of numerical stability, particularly in the log-variance term of the KL loss [S1, unofficial].
*   **Boundaries:** The spherical folding grid assumes a specific topology; points near the boundaries of the grid may require special handling to ensure smooth interpolation [S1, unofficial].
*   **Degenerate Input:** If the input point cloud is sparse or noisy, the encoder may produce poor latent representations, leading to inaccurate Gaussian initialization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the depth of the encoder and decoder networks and the size of the spherical folding grid. The Chamfer loss computation is $O(N \cdot M)$ where $N$ and $M$ are the number of points in the input and reconstructed sets, respectively [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of parameters in the neural network and the size of the grid ($grid\_dim^2$) [S1, unofficial].
*   **Strengths:** Provides a data-driven initialization that can capture complex geometric structures better than heuristic methods [S1, unofficial].
*   **Limitations:** Requires training data and computational resources for the VAE training phase [S1, unofficial].
*   **When to use:** Suitable for scenes where high-quality initialization is critical and training data is available [S1, unofficial].
*   **When not to use:** Not suitable for real-time applications where pre-training is not feasible [S1, unofficial].

## Background
SF-VAE was introduced as part of the "gs-embedding" project, an unofficial implementation exploring the integration of VAEs with 3D Gaussian Splatting [S1, unofficial]. It builds upon the principles of learning-based initialization for Gaussian Splatting, aiming to improve the robustness of the initialization process by leveraging learned priors from point cloud data [S1, unofficial].

## Sources
[S1] cilix-ai/gs-embedding, https://github.com/cilix-ai/gs-embedding, fetched 2026-10-27 (unofficial)
