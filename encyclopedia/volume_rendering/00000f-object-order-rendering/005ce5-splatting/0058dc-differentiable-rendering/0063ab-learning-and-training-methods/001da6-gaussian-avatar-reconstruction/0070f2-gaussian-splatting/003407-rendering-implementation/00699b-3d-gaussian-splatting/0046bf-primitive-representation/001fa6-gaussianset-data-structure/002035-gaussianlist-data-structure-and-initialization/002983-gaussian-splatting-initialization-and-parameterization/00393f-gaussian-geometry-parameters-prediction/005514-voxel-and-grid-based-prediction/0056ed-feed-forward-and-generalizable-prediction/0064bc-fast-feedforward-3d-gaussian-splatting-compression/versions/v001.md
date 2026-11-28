# Fast Feedforward 3D Gaussian Splatting Compression


## Overview
Fast Feedforward 3D Gaussian Splatting Compression (FCGS) is an autoencoder-based compression method that encodes 3D Gaussian attributes into a latent space using hyperpriors and context models, addressing the fidelity drop caused by the non-invertibility of MLPs in geometry attributes [S1, unofficial]. Unlike anchor-based quantization methods that modify Gaussian locations and require finetuning, FCGS creates context grids in a feed-forward manner without altering the original Gaussian positions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G}$, where each Gaussian $i$ has geometry attributes $f_{geo} \in \mathbb{R}^8$ (opacity $\alpha$, diagonal scaling $S$, quaternion rotation $R$) and color attributes $f_{col} \in \mathbb{R}^{48}$ (3-degree Spherical Harmonics coefficients). The concatenated attribute vector is $f_{gau} \in \mathbb{R}^{56}$ [S1, unofficial].
*   **Output:** A compressed bitstream representing the latent variables $\hat{y}$ and hypervariables $\hat{z}$, which can be decoded back to reconstructed Gaussian attributes $\hat{x}$ [S1, unofficial].

**Data Structures:**
*   **Transform Networks:**
    *   Analysis transform ($ga$) and Synthesis transform ($gs$): Multi-Layer Perceptrons (MLPs) with 4 layers each [S1, unofficial].
    *   Hyperprior Analysis ($ha$) and Synthesis ($hs$): MLPs with 3 layers each [S1, unofficial].
*   **Context Models:**
    *   Inter-Gaussian Context: Uses 3D and 2D grids at multiple resolutions to store features $f_{kv}$ from previously decoded Gaussians [S1, unofficial].
    *   Intra-Gaussian Context: Processes chunks of the latent vector $\hat{y}_i$ sequentially [S1, unofficial].
*   **Probability Model:** Gaussian Mixture Model (GMM) combining parameters from hyperprior, inter-context, and intra-context models [S1, unofficial].

**Parameters:**
*   $N_s$: Number of batches for inter-Gaussian context decoding. Typical value: 2 [S1, unofficial].
*   $N_c$: Number of chunks for intra-Gaussian context. $N_c=4$ for path $m=1$ (color via MLP), $N_c=3$ for path $m=0$ (color bypass) [S1, unofficial].
*   $q$: Quantization step size. $q=1$ if $m_i=1$ (latent space), $q$ is a trainable decimal parameter if $m_i=0$ (bypass) [S1, unofficial].
*   $\epsilon_m$: Hyperparameter threshold for binarizing the mask $m$ [S1, unofficial].
*   $\lambda_m$: Explicitly NOT used in FCGS to avoid complicating Rate-Distortion (RD) trade-off [S1, unofficial].

**Algorithm Steps:**

1.  **Attribute Partitioning:** Separate Gaussian attributes into geometry $f_{geo}$ and color $f_{col}$. Concatenate to form $f_{gau}$ [S1, unofficial].
2.  **Mask Generation (MEM):** For each Gaussian $i$, compute a binary mask $m_i$ to determine if color attributes pass through the MLPs or bypass them:
    $$ m_i = \text{Sig}(\text{MLP}_m(f_{gau})) > \epsilon_m $$
    where $\text{Sig}$ is the Sigmoid function [S1, unofficial].
3.  **Latent Encoding:**
    *   For geometry attributes ($f_{geo}$): Bypass MLPs to ensure exact recovery. Set $y = x$ [S1, unofficial].
    *   For color attributes ($f_{col}$): Apply the Mixture Model (MEM) logic. Compute the latent $y_i$:
        $$ \hat{y}_i = gs(Q(ga(x_i))) \times m_i + Q(x_i) \times (1 - m_i) $$
        where $Q$ is quantization with step $q$ (1 if $m_i=1$, trainable decimal if $m_i=0$) [S1, unofficial].
    *   Compute hyperprior $z_i = ha(x_i)$ [S1, unofficial].
4.  **Inter-Gaussian Context Decoding:**
    *   Split all Gaussians' latents $\hat{y}$ into $N_s$ batches. Decode batches sequentially [S1, unofficial].
    *   For a Gaussian $i$ in the current batch, located at $\mu_{gi}$, identify the set of voxels $V_{\mu_i}^g$ forming its minimum bounding box in the grid [S1, unofficial].
    *   Compute interpolated feature $f_{i\mu}$:
        $$ f_{i\mu} = \sum_{k: v_k \in V_{\mu_i}^g} w_k f_{kv} $$
        where $w_k$ are interpolation weights [S1, unofficial].
    *   Compute inter-context distribution parameters $\mu_{si}, \sigma_{is}, \pi_{is}$:
        $$ \mu_{si}, \sigma_{is}, \pi_{is} = \text{MLP}_s(\oplus [f_{i\mu}, \text{emb}(\mu_{gi})]) $$
        where $\text{emb}$ is sincos positional embedding and $\oplus$ is channel-wise concatenation [S1, unofficial].
5.  **Intra-Gaussian Context Decoding:**
    *   Split $\hat{y}_i$ into $N_c$ chunks.
    *   For path $m=1$, split along channel dimension ($N_c=4$). For path $m=0$, split along RGB axis ($N_c=3$) [S1, unofficial].
    *   Compute intra-context parameters for chunk $n_c$:
        $$ \mu_{i,n_c}, \sigma_{i,n_c}, \pi_{i,n_c} = \text{MLP}_c(\hat{y}_{i,[0, n_c-1]}) $$
    *   Aggregate parameters:
        $$ \mu_{ci}, \sigma_{ic}, \pi_{ic} = \oplus_{n_c=1}^{N_c} \{\mu_{i,n_c}, \sigma_{i,n_c}, \pi_{i,n_c}\} $$
        [S1, unofficial].
6.  **Hyperprior Context:** Compute hyperprior parameters $\mu_{hi}, \sigma_{ih}, \pi_{ih}$ from $\hat{z}_i$ using $hs$ [S1, unofficial].
7.  **Probability Estimation:** Combine inter-context, intra-context, and hyperprior parameters using a Gaussian Mixture Model (GMM) to estimate the probability of each latent value $\hat{y}_i$ [S1, unofficial].
8.  **Arithmetic Decoding:** Use the estimated probabilities to decode the bitstream into $\hat{y}$ and $\hat{z}$ [S1, unofficial].
9.  **Reconstruction:** Apply synthesis transform $gs$ to $\hat{y}$ (if $m_i=1$) or use $\hat{y}$ directly (if $m_i=0$) to obtain $\hat{x}$ [S1, unofficial].

**Pseudocode:**

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FCGSCompression:
    def __init__(self, num_gaussians, num_channels_geo=8, num_channels_col=48):
        self.num_gaussians = num_gaussians
        self.num_channels_geo = num_channels_geo
        self.num_channels_col = num_channels_col
        self.num_channels_gau = num_channels_geo + num_channels_col
        
        # Transform Networks (MLPs)
        self.ga = MLP(layers=4, in_dim=self.num_channels_gau, out_dim=128) # Example latent dim
        self.gs = MLP(layers=4, in_dim=128, out_dim=self.num_channels_gau)
        self.ha = MLP(layers=3, in_dim=self.num_channels_gau, out_dim=64) # Example hyper dim
        self.hs = MLP(layers=3, in_dim=64, out_dim=128)
        
        # Context Models
        self.mlp_s = MLP(layers=2, in_dim=128 + 6, out_dim=128) # 128 latent + 6 pos embed
        self.mlp_c = MLP(layers=2, in_dim=128, out_dim=128)
        self.mlp_m = MLP(layers=2, in_dim=self.num_channels_gau, out_dim=1)
        
        # Grids for Inter-Gaussian Context
        self.grid_3d = None
        self.grid_2d = None
        
        # Parameters
        self.N_s = 2
        self.N_c = 4 # Default for m=1
        self.epsilon_m = 0.5
        self.q_trainable = nn.Parameter(torch.tensor(0.1))

    def encode(self, x_geo, x_col):
        """
        x_geo: [N, 8]
        x_col: [N, 48]
        """
        x = torch.cat([x_geo, x_col], dim=1) # [N, 56]
        
        # Mask Generation
        mask_logits = self.mlp_m(x)
        mask = (torch.sigmoid(mask_logits) > self.epsilon_m).float() # [N, 1]
        
        # Latent Encoding
        # Geometry: Bypass
        y_geo = x_geo
        
        # Color: MEM
        x_col_encoded = self.ga(x_col)
        x_col_quantized = self.quantize(x_col, q=1.0) # Bypass case quantization step
        x_col_mlp = self.quantize(self.gs(x_col_encoded), q=1.0) # MLP case quantization step
        
        # Note: The formula uses Q(ga(x)) inside gs, but effectively it's:
        # y = gs(Q(ga(x))) * m + Q(x) * (1-m)
        # However, Q is applied before or after? The text says:
        # x_hat = gs(Q(ga(x))) * m + Q(x) * (1-m)
        # This implies y is the input to gs? No, y is the latent.
        # Let's follow the formula strictly:
        # y_i = gs(Q(ga(x_i))) * m_i + Q(x_i) * (1 - m_i)
        # But Q is quantization. In compression, we quantize the latent y.
        # The text says: "if y_i is in the latent space (m_i=1), q is 1; otherwise (m_i=0), q becomes a trainable decimal parameter."
        # And "x_hat = gs(Q(ga(x))) * m + Q(x) * (1-m)"
        # This suggests the quantization happens on the output of the transforms or the input?
        # Actually, standard AE: x -> ga -> y -> Q -> y_hat -> gs -> x_hat.
        # FCGS modifies this:
        # If m=1: y = Q(ga(x)). Then x_hat = gs(y).
        # If m=0: y = Q(x). Then x_hat = y (bypass gs? No, text says "simply bypass the MLPs... setting y=x").
        # Wait, if y=x, then x_hat = gs(y) = gs(x)? No, "bypass the MLPs... resulting in y=ga(x) or y=x".
        # If y=x, then we don't use gs? The formula says x_hat = gs(...) * m + Q(x) * (1-m).
        # This implies if m=0, x_hat = Q(x). So gs is not used for m=0.
        
        # Re-reading: "x_hat_i = gs(Q(ga(x_i))) * m_i + Q(x_i) * (1 - m_i)"
        # This means:
        # If m_i=1: x_hat = gs(Q(ga(x)))
        # If m_i=0: x_hat = Q(x)
        
        # So y is not directly x_hat. y is the latent to be compressed.
        # If m=1: y = Q(ga(x)). Compress y. Decode y_hat. x_hat = gs(y_hat).
        # If m=0: y = Q(x). Compress y. Decode y_hat. x_hat = y_hat.
        
        # Let's implement the latent generation for compression
        y = torch.zeros_like(x)
        for i in range(self.num_gaussians):
            if mask[i] > 0.5:
                # Path 1: MLP
                y_latent = self.ga(x[i])
                y[i] = self.quantize(y_latent, q=1.0)
            else:
                # Path 0: Bypass
                y[i] = self.quantize(x[i], q=self.q_trainable)
                
        # Hyperprior
        z = self.ha(x)
        z_quantized = self.quantize(z, q=1.0)
        
        return y_quantized, z_quantized, mask

    def decode(self, y_hat, z_hat, grid_features, gaussian_positions):
        """
        y_hat: [N, 56] (decoded latent)
        z_hat: [N, 64] (decoded hyperprior)
        grid_features: 3D/2D grid structures
        gaussian_positions: [N, 3]
        """
        x_hat = torch.zeros_like(y_hat)
        
        # Split into batches for Inter-Gaussian Context
        batch_size = self.num_gaussians // self.N_s
        indices = torch.randperm(self.num_gaussians)
        
        for ns in range(self.N_s):
            start_idx = ns * batch_size
            end_idx = (ns + 1) * batch_size
            current_indices = indices[start_idx:end_idx]
            
            for idx in current_indices:
                # Inter-Gaussian Context
                mu_gi = gaussian_positions[idx]
                # Get grid features via interpolation
                f_i_mu = self.interpolate_grid(grid_features, mu_gi)
                pos_emb = self.sincos_pos_embed(mu_gi)
                context_params_inter = self.mlp_s(torch.cat([f_i_mu, pos_emb], dim=-1))
                
                # Intra-Gaussian Context
                y_i = y_hat[idx]
                # Split into chunks
                if self.N_c == 4:
                    chunks = torch.chunk(y_i, 4, dim=-1)
                else:
                    # Split along RGB (assuming 3 channels for color part? No, 48 channels)
                    # Text says "split along RGB axis" for m=0. 
                    # For m=1, split along channel.
                    # We assume standard channel split for simplicity here as m=0 bypasses MLP
                    chunks = torch.chunk(y_i, 3, dim=-1) 
                
                context_params_intra = []
                for nc in range(len(chunks)):
                    prev_chunks = chunks[:nc]
                    if prev_chunks:
                        prev_y = torch.cat(prev_chunks, dim=-1)
                    else:
                        prev_y = torch.zeros_like(chunks[0])
                    
                    params = self.mlp_c(prev_y)
                    context_params_intra.append(params)
                
                # Aggregate intra-context
                context_params_intra = torch.cat(context_params_intra, dim=-1)
                
                # Hyperprior Context
                z_i = z_hat[idx]
                context_params_hyper = self.hs(z_i)
                
                # GMM Probability Estimation (Simplified)
                # Combine params to get mean/var for quantization error correction
                # This step is complex and depends on the specific GMM implementation
                # For the purpose of reconstruction, we assume the decoder uses these params
                # to refine the probability estimate for arithmetic decoding.
                # Since we already have y_hat, we just need to reconstruct x_hat.
                
                # Reconstruct x_hat
                # If the latent was from MLP path, apply gs
                # If bypass, y_hat is already x_hat (quantized)
                # We need to know the mask to decide. 
                # In a real system, the mask is also compressed or derived.
                # Assuming mask is known or compressed separately.
                # For this pseudocode, we assume we can determine the path.
                
                # Simplified reconstruction:
                # x_hat[idx] = self.gs(y_hat[idx]) # If MLP path
                # But we need to know if it was MLP or Bypass.
                # Let's assume we decode the mask first or it's part of the stream.
                
                # For now, just return y_hat as a placeholder for x_hat
                # In reality, x_hat = gs(y_hat) if m=1 else y_hat
                pass
                
        return x_hat

    def quantize(self, x, q):
        return torch.round(x / q) * q

    def interpolate_grid(self, grid, pos):
        # Placeholder for grid interpolation
        return torch.zeros(128)

    def sincos_pos_embed(self, pos):
        # Placeholder for positional embedding
        return torch.zeros(6)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians $N$ for the feed-forward encoding/decoding steps, plus the cost of grid interpolation for context models.
*   **Memory Complexity:** Depends on the grid size for inter-Gaussian context. The grids store features for all Gaussians, requiring $O(N)$ space if dense, or less if sparse.
*   **Performance:** FCGS achieves high compression ratios while maintaining rendering fidelity. It avoids the fidelity drop associated with MLP non-invertibility by bypassing geometry attributes [S1, unofficial].
*   **Strengths:** Feed-forward nature allows fast inference without per-scene optimization. Context models improve probability estimation accuracy.
*   **Limitations:** Requires training on a large dataset. Grid creation and interpolation add computational overhead compared to simple quantization [S1, unofficial].

## Background
FCGS was introduced in 2024 by authors associated with the ICLR 2025 conference paper "Fast Feedforward 3D Gaussian Splatting Compression" [S1, unofficial]. It builds upon the principles of 3D Gaussian Splatting (3DGS) [Kerbl et al., 2023] and image compression techniques, specifically autoencoders with hyperpriors and context models. It addresses the limitations of previous compression methods that suffered from fidelity drops due to the non-invertibility of MLPs in geometry attributes [S1, unofficial].

## Sources
[S1] Fast Feedforward 3D Gaussian Splatting Compression, https://doi.org/10.48550/arxiv.2410.08017, fetched 2026-10-26 (unofficial)
