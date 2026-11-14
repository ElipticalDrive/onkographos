# TransNeRF

Also called: Generalizable Neural Radiance Fields for Novel View Synthesis with Transformer

## Overview
TransNeRF is a generalizable neural radiance field method that learns a generic 3D scene representation from multiple source views without requiring per-scene optimization [S1, unofficial]. It replaces the pooling-based multi-view aggregation of prior methods with an encoder-decoder Transformer framework that explores deep relationships among views and enforces local 5D radiance field consistency via attention mechanisms [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $M$ source views $\{I_m, \Theta_m\}_{m=1}^M$, where $I_m$ is the image and $\Theta_m$ contains camera pose, intrinsics, and scene bounds. A query 3D point $(x, y, z)$ and its viewing direction $d$ on a target ray.
*   **Outputs:** Volumetric density $\sigma$ and directional color $c$ for the query point.

**Data Structures:**
*   **Feature Volume:** Extracted from a pre-trained U-Net for each source view.
*   **Embeddings:** Learnable query embeddings and source-view embeddings are concatenated.
*   **Attention Heads:** Multi-head attention uses $H$ heads, where $H \times d_h = d_k$. Queries $Q \in \mathbb{R}^{N_q \times d_k}$, Keys $K \in \mathbb{R}^{N_{kv} \times d_k}$, Values $V \in \mathbb{R}^{N_{kv} \times d_v}$.

**Steps:**

1.  **Feature Extraction:** For each source view $m$, extract a feature volume using a pre-trained U-Net. Retrieve spatially aligned 2D-pixel features for the query 3D point $(x, y, z)$ projected into view $m$. Record the source viewing direction $d^m_{src}$ for the projected pixel. Compute initial source-view embeddings $\{x^0_m\}_{m=1}^M$ [S1, unofficial].
2.  **Density-ViewDecoder (Surrounding-view Space):**
    *   Input $X_0$ is the concatenation of a learnable query density embedding $x^0_\sigma$ and the initial source-view embeddings: $X_0 = [x^0_\sigma; x^0_1; \dots; x^0_M]$ [S1, unofficial].
    *   Process through $L$ blocks. Each block $l$ applies:
        $$ \tilde{X}_l = \text{Norm}(\text{Density-ViAttn}(X_{l-1}) + X_{l-1}) $$
        $$ X_l = \text{Norm}(\text{FFN}(\tilde{X}_l) + \tilde{X}_l) $$
    *   $\text{Density-ViAttn}(X) = \text{MH-Attn}(X, X, X)$ [S1, unofficial].
    *   Output $X_L = [x^L_\sigma; x^L_1; \dots; x^L_M]$ [S1, unofficial].
3.  **Density-RayDecoder (Ray-cast Space):**
    *   Compute initial density representation for the query point: $\sigma^0 = F_C(x^L_\sigma(x, y, z))$, where $F_C$ is a linear function [S1, unofficial].
    *   Consider $2n+1$ neighboring points along the target ray: $\{q_{i-n}, \dots, q_i, \dots, q_{i+n}\}$.
    *   Input $D_0$ is the concatenation of initial density representations of these neighbors plus positional encodings $E_{pos}$: $D_0 = [\sigma^0_{i-n}; \dots; \sigma^0_{i+n}] + E_{pos}$ [S1, unofficial].
    *   Process through $L$ blocks:
        $$ \tilde{D}_l = \text{Norm}(\text{Density-Attn}(D_{l-1}) + D_{l-1}) $$
        $$ D_l = \text{Norm}(\text{FFN}(\tilde{D}_l) + D_{l-1}) $$
    *   $\text{Density-Attn}(D) = \text{MH-Attn}(D, D, D)$ [S1, unofficial].
    *   Output the final density representation $\sigma^L_i$ and project to density value $\sigma$ [S1, unofficial].
4.  **Color-ViewDecoder (Surrounding-view Space):**
    *   Input Query $Y_0 = F_C(\sigma^L_i) \oplus d_{tgt}$, where $d_{tgt}$ is the target viewing direction [S1, unofficial].
    *   Keys $X_{src} = [F_C(x^L_1) \oplus d^1_{src}; \dots; F_C(x^L_M) \oplus d^M_{src}]$ [S1, unofficial].
    *   Values $C_{src} = [\gamma(c^1_{src}); \dots; \gamma(c^M_{src})]$, where $\gamma$ is an embedding function and $c^m_{src}$ is the source view RGB color [S1, unofficial].
    *   Process through $L$ blocks:
        $$ \tilde{Y}_l = \text{Norm}(\text{Color-ViAttn}(Y_{l-1}, X_{src}, C_{src}) + Y_{l-1}) $$
        $$ Y_l = \text{Norm}(\text{FFN}(\tilde{Y}_l) + \tilde{Y}_l) $$
    *   $\text{Color-ViAttn}(Y, X_{src}, C_{src}) = \text{MH-Attn}(Y, X_{src}, C_{src})$ [S1, unofficial].
    *   Output latent color representation $y^L_c$ [S1, unofficial].
5.  **Color-RayDecoder (Ray-cast Space):**
    *   Assign $y^L_c$ to the corresponding neighbor embedding $z^0_i$.
    *   Input $Z_0 = [z^0_{i-n}; \dots; z^0_{i+n}] + E_{pos}$ [S1, unofficial].
    *   Process through $L$ blocks:
        $$ \tilde{Z}_l = \text{Norm}(\text{Color-Attn}(Z_{l-1}) + Z_{l-1}) $$
        $$ Z_l = \text{Norm}(\text{FFN}(\tilde{Z}_l) + Z_{l-1}) $$
    *   $\text{Color-Attn}(Z) = \text{MH-Attn}(Z, Z, Z)$ [S1, unofficial].
    *   Project final embedding $z^L_i$ to color $c$ [S1, unofficial].
6.  **Rendering:** Use classical volume rendering to compute pixel color $C(r)$ for ray $r(t) = o + td$:
    $$ C(r) = \int_{t_n}^{t_f} T(t)\sigma(r(t))c(r(t), d)dt, \quad T(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s))ds\right) $$
    [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class TransNeRF:
    def __init__(self, num_heads, head_dim, num_layers, num_neighbors, num_source_views):
        self.num_heads = num_heads
        self.head_dim = head_dim
        self.num_layers = num_layers
        self.num_neighbors = num_neighbors
        self.num_source_views = num_source_views
        
        # Learnable embeddings
        self.query_density_emb = nn.Parameter(torch.randn(1, 1, 128)) # Example dim
        self.query_color_emb = nn.Parameter(torch.randn(1, 1, 128))
        
        # Linear projections
        self.fc_density = nn.Linear(128, 1)
        self.fc_color = nn.Linear(128, 3)
        self.fc_proj = nn.Linear(128, 128)
        
        # Attention modules (simplified)
        self.density_view_attn = MultiHeadAttention(num_heads, head_dim)
        self.density_ray_attn = MultiHeadAttention(num_heads, head_dim)
        self.color_view_attn = MultiHeadAttention(num_heads, head_dim)
        self.color_ray_attn = MultiHeadAttention(num_heads, head_dim)
        
        # FFNs
        self.ffn_density_view = FeedForwardNetwork(128)
        self.ffn_density_ray = FeedForwardNetwork(128)
        self.ffn_color_view = FeedForwardNetwork(128)
        self.ffn_color_ray = FeedForwardNetwork(128)
        
        # Positional Encoding
        self.pos_enc = PositionalEncoding(128)

    def forward(self, source_views, query_point, query_direction):
        # source_views: list of (image, pose, intrinsics)
        # query_point: (3,)
        # query_direction: (3,)
        
        # Step 1: Feature Extraction
        initial_source_embs = []
        source_directions = []
        source_colors = []
        
        for view in source_views:
            # Extract features using pre-trained U-Net (not implemented here)
            # Retrieve 2D-pixel features for query_point projected into view
            feat = extract_u_net_feature(view, query_point) 
            initial_source_embs.append(feat)
            source_directions.append(get_view_direction(view, query_point))
            source_colors.append(get_pixel_color(view, query_point))
            
        initial_source_embs = torch.stack(initial_source_embs, dim=0) # (M, 1, D)
        
        # Step 2: Density-ViewDecoder
        X0 = torch.cat([self.query_density_emb, initial_source_embs], dim=1) # (M+1, 1, D)
        X = X0
        for l in range(self.num_layers):
            X_attn = self.density_view_attn(X, X, X)
            X = X + X_attn
            X = F.layer_norm(X, X.shape[-1:])
            X_ffn = self.ffn_density_view(X)
            X = X + X_ffn
            X = F.layer_norm(X, X.shape[-1:])
            
        # Extract query density embedding
        x_sigma_L = X[0] # First element is query
        
        # Step 3: Density-RayDecoder
        # Get neighbors along ray
        neighbors = get_ray_neighbors(query_point, query_direction, self.num_neighbors)
        initial_densities = []
        for n in neighbors:
            # Project query point embedding to density
            # Note: In practice, we need features for neighbors too. 
            # Simplified: assume we can get features for neighbors similarly
            feat_n = extract_u_net_feature(source_views[0], n) # Placeholder for neighbor features
            # Actually, the paper says initial density rep is linear function of Density-ViewDecoder output
            # We need to run Density-ViewDecoder for each neighbor? 
            # The paper implies we compute sigma^0 for each neighbor.
            # For implementation, we would batch process neighbors through the view decoder.
            # Here we assume a helper function returns the projected density for a point
            sigma_0_n = self.fc_density(self.fc_proj(feat_n)).squeeze() 
            initial_densities.append(sigma_0_n)
            
        D0 = torch.stack(initial_densities) # (2n+1, D)
        D0 = D0 + self.pos_enc(D0)
        
        D = D0
        for l in range(self.num_layers):
            D_attn = self.density_ray_attn(D, D, D)
            D = D + D_attn
            D = F.layer_norm(D, D.shape[-1:])
            D_ffn = self.ffn_density_ray(D)
            D = D + D_ffn
            D = F.layer_norm(D, D.shape[-1:])
            
        # Output density for the query point (center neighbor)
        sigma = self.fc_density(D[self.num_neighbors]).squeeze()
        
        # Step 4: Color-ViewDecoder
        # Query Y0
        Y0 = self.fc_density(self.fc_proj(x_sigma_L)) + query_direction # Concat/Sum? Paper says concat then FC
        # Correction: Y0 = FC(sigma_L) + d_tgt. Assuming FC projects sigma_L to match dim of d_tgt or vice versa.
        # Paper: Y0 = FC(sigma_L^i) + d_tgt. Let's assume FC makes them compatible.
        Y0 = self.fc_color(self.fc_proj(x_sigma_L)) + query_direction 
        
        # Keys and Values
        # Keys: FC(x_L^m) + d_src^m
        keys = []
        for m in range(self.num_source_views):
            x_m_L = X[m+1] # Skip query
            key = self.fc_proj(x_m_L) + source_directions[m]
            keys.append(key)
        K = torch.stack(keys, dim=0) # (M, D)
        
        # Values: gamma(c_src^m)
        values = []
        for m in range(self.num_source_views):
            val = self.fc_color(torch.tensor(source_colors[m])) # Embedding function
            values.append(val)
        V = torch.stack(values, dim=0) # (M, D)
        
        Y = Y0.unsqueeze(0) # (1, D)
        for l in range(self.num_layers):
            Y_attn = self.color_view_attn(Y, K, V)
            Y = Y + Y_attn
            Y = F.layer_norm(Y, Y.shape[-1:])
            Y_ffn = self.ffn_color_view(Y)
            Y = Y + Y_ffn
            Y = F.layer_norm(Y, Y.shape[-1:])
            
        y_c_L = Y[0]
        
        # Step 5: Color-RayDecoder
        # Assign y_c_L to center neighbor
        initial_colors = []
        for n in neighbors:
            # Get features for neighbor to compute color rep?
            # Paper: latent color rep y_c^L is assigned to z_i^0
            # We need to compute y_c^L for each neighbor.
            # Simplified: assume we have a batch of y_c_L for all neighbors
            # Here we just use the center one for the query point
            pass
            
        # For the query point, we use the center neighbor's color rep
        z0 = y_c_L.unsqueeze(0) # (1, D)
        Z0 = z0 + self.pos_enc(z0)
        
        Z = Z0
        for l in range(self.num_layers):
            Z_attn = self.color_ray_attn(Z, Z, Z)
            Z = Z + Z_attn
            Z = F.layer_norm(Z, Z.shape[-1:])
            Z_ffn = self.ffn_color_ray(Z)
            Z = Z + Z_ffn
            Z = F.layer_norm(Z, Z.shape[-1:])
            
        c = self.fc_color(Z[0]).squeeze()
        
        return sigma, c

class MultiHeadAttention(nn.Module):
    def __init__(self, num_heads, head_dim):
        super().__init__()
        self.num_heads = num_heads
        self.head_dim = head_dim
        self.scale = head_dim ** -0.5
        
    def forward(self, Q, K, V):
        # Q, K, V: (N, D)
        N, D = Q.shape
        H = self.num_heads
        dh = self.head_dim
        
        # Reshape to (N, H, dh)
        Q = Q.view(N, H, dh).transpose(0, 1) # (H, N, dh)
        K = K.view(N, H, dh).transpose(0, 1)
        V = V.view(N, H, dh).transpose(0, 1)
        
        # Attention
        scores = torch.matmul(Q, K.transpose(-2, -1)) * self.scale # (H, N, N)
        attn = F.softmax(scores, dim=-1)
        out = torch.matmul(attn, V) # (H, N, dh)
        
        # Concatenate heads
        out = out.transpose(0, 1).reshape(N, H * dh) # (N, D)
        return out

class FeedForwardNetwork(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.fc1 = nn.Linear(dim, dim * 4)
        self.fc2 = nn.Linear(dim * 4, dim)
        
    def forward(self, x):
        return self.fc2(F.gelu(self.fc1(x)))

class PositionalEncoding(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.dim = dim
        
    def forward(self, x):
        # Simplified PE
        return x
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of source views $M$, the number of attention heads $H$, and the sequence length (neighbors + views). Attention is $O(M \cdot N_{neigh}^2)$ or similar depending on implementation.
*   **Memory Complexity:** Stores feature volumes for all source views and embeddings for all neighbors and views.
*   **Performance:** Generalizes to unseen scenes without per-scene finetuning [S1, unofficial].
*   **Strengths:** Explores deep relationships among multiple views; enforces local 5D radiance field consistency [S1, unofficial].
*   **Limitations:** Requires a large dataset of diverse scenes for training; pre-trained U-Net dependency [S1, unofficial].

## Background
TransNeRF was proposed to address the limitations of pooling-based multi-view representations that ignore local consistency and complex relationships across views [S1, unofficial]. It builds on the Transformer architecture's success in NLP and CV [S1, unofficial]. It is used for novel view synthesis, generalizing to unseen scenes [S1, unofficial].

## Sources
[S1] Generalizable Neural Radiance Fields for Novel View Synthesis with Transformer, https://doi.org/10.48550/arxiv.2206.05375, fetched 2026-10-27 (unofficial)
