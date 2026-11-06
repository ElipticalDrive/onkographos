# Epipolar Encoder

Also called: Epipolar Transformer, Multi-View Epipolar Encoder

## Overview
The Epipolar Encoder is a two-view (or multi-view) neural module that resolves scale ambiguity in 3D reconstruction by aggregating cross-view features along epipolar lines. It solves the problem where camera poses are scaled by an arbitrary factor $s_i$, preventing the inference of metric depth from a single view. Unlike standard volume rendering which samples along a ray, this method samples features from a reference pixel and correlates them with features sampled along the corresponding epipolar line in a second view, augmented with positionally encoded depths. This allows the encoder to memorize the correct depth consistent with the scene's arbitrary scale, which is then used for subsequent Gaussian parameter prediction.

## How it works
The method operates on a pair of reference views, denoted as $I$ (reference) and $\tilde{I}$ (secondary), though it extends to $N$ views. The core mechanism is epipolar cross-attention, which aligns features from $I$ with potential correspondences in $\tilde{I}$ using geometric depth constraints.

### Inputs and Outputs
*   **Inputs:**
    *   Reference image $I$ and secondary image $\tilde{I}$.
    *   Camera poses for $I$ and $\tilde{I}$ (extrinsics $E, \tilde{E}$ and intrinsics $K, \tilde{K}$).
    *   Feature volumes $\mathbf{F}$ and $\tilde{\mathbf{F}}$ of shape $(H, W, C)$, where $H, W$ are image dimensions and $C$ is the feature dimension. These are produced by a per-image feature encoder (e.g., a CNN or ViT) applied to $I$ and $\tilde{I}$ separately.
*   **Outputs:**
    *   Updated feature volume $\mathbf{F}'$ of shape $(H, W, C)$, where each pixel $\mathbf{F}'[\mathbf{u}]$ contains aggregated cross-view information and encoded depth consistent with the scene scale.

### Data Structures
*   **Feature Volumes:** $\mathbf{F}, \tilde{\mathbf{F}} \in \mathbb{R}^{H \times W \times C}$.
*   **Embedding Matrices:** $\mathbf{Q}, \mathbf{K}, \mathbf{V} \in \mathbb{R}^{C \times C_{head}}$. These are learnable linear projection matrices.
*   **Positional Encoding:** $\gamma: \mathbb{R} \to \mathbb{R}^{C_{enc}}$. A standard positional encoding function (e.g., sine/cosine or learned) mapping a scalar depth to a vector.
*   **Per-Image Embeddings:** $\mathbf{e}_i \in \mathbb{R}^{C_{emb}}$ for each view $i$ (used in multi-view variant).

### Numbered Steps

1.  **Per-Image Encoding:** Encode each input view into a feature volume.
    $$ \mathbf{F} = \text{Encoder}(I), \quad \tilde{\mathbf{F}} = \text{Encoder}(\tilde{I}) $$
    [G] The encoder is typically a shared-weight convolutional or transformer backbone.

2.  **Epipolar Line Computation:** For each pixel $\mathbf{u} = (u_x, u_y)$ in $I$:
    *   Compute the 3D ray direction $\mathbf{r}$ in the camera coordinate frame of $I$ using $\mathbf{u}$ and $K$.
    *   Project this ray onto the image plane of $\tilde{I}$ to obtain the epipolar line $\ell$.
    *   Mathematically, if $\mathbf{u}$ corresponds to ray $\mathbf{r}$, the epipolar line $\ell$ in $\tilde{I}$ is the projection of the ray originating at $I$'s camera center and passing through $\mathbf{u}$.

3.  **Epipolar Sampling:** Sample pixel coordinates $\{\tilde{\mathbf{u}}_l\}$ along the epipolar line $\ell$ in $\tilde{I}$.
    *   Let $L$ be the number of samples along the line.
    *   $\{\tilde{\mathbf{u}}_l\}_{l=1}^L$ are 2D coordinates in $\tilde{I}$.

4.  **Depth Triangulation:** For each sample $\tilde{\mathbf{u}}_l$:
    *   Compute the 3D point $\mathbf{X}_l$ by triangulating the ray from $I$ through $\mathbf{u}$ and the ray from $\tilde{I}$ through $\tilde{\mathbf{u}}_l$.
    *   Compute the depth $\tilde{d}_{\tilde{\mathbf{u}}_l}$ as the distance from $\mathbf{X}_l$ to $I$'s camera origin.
    *   [G] If the rays are parallel or nearly parallel, triangulation may be unstable; in practice, a minimum distance threshold or regularization is applied.

5.  **Feature Augmentation:** For each sample $\tilde{\mathbf{u}}_l$, create an augmented feature vector $\mathbf{s}$:
    $$ \mathbf{s} = \tilde{\mathbf{F}}[\tilde{\mathbf{u}}_l] \oplus \gamma(\tilde{d}_{\tilde{\mathbf{u}}_l}) $$
    where $\oplus$ denotes concatenation. The depth $\tilde{d}_{\tilde{\mathbf{u}}_l}$ encodes the metric scale information.

6.  **Attention Projection:** Compute query, key, and value vectors for the epipolar attention:
    $$ \mathbf{q} = \mathbf{Q} \cdot \mathbf{F}[\mathbf{u}] $$
    $$ \mathbf{k}_l = \mathbf{K} \cdot \mathbf{s} $$
    $$ \mathbf{v}_l = \mathbf{V} \cdot \mathbf{s} $$
    for each $l \in \{1, \dots, L\}$.

7.  **Epipolar Cross-Attention:** Update the feature $\mathbf{F}[\mathbf{u}]$ using softmax attention over the epipolar samples:
    $$ \mathbf{F}[\mathbf{u}] \leftarrow \mathbf{F}[\mathbf{u}] + \text{Att}(\mathbf{q}, \{\mathbf{k}_l\}, \{\mathbf{v}_l\}) $$
    where $\text{Att}(\mathbf{q}, \{\mathbf{k}\}, \{\mathbf{v}\}) = \sum_{l} \text{softmax}_l \left( \frac{\mathbf{q} \cdot \mathbf{k}_l}{\sqrt{C_{head}}} \right) \mathbf{v}_l$.
    The skip connection ($+$) ensures the original feature is preserved. The correct correspondence (where $\tilde{\mathbf{u}}_l$ matches the true 3D point) will receive the highest attention weight, effectively encoding the scaled depth into $\mathbf{F}[\mathbf{u}]$.

8.  **Per-Image Self-Attention:** Apply a self-attention layer to the entire feature volume $\mathbf{F}$ to propagate depth estimates to pixels without epipolar correspondences (e.g., occluded regions or textureless areas):
    $$ \mathbf{F} \leftarrow \mathbf{F} + \text{SelfAttention}(\mathbf{F}) $$
    [G] SelfAttention is a standard multi-head self-attention mechanism applied to the flattened or spatially arranged features of $\mathbf{F}$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class EpipolarEncoder(nn.Module):
    def __init__(self, feature_dim, head_dim=64, num_heads=8, num_samples=64):
        super().__init__()
        self.num_samples = num_samples
        self.num_heads = num_heads
        self.head_dim = head_dim
        
        # Projection matrices for cross-attention
        self.q_proj = nn.Linear(feature_dim, head_dim * num_heads)
        self.k_proj = nn.Linear(feature_dim + 64, head_dim * num_heads) # +64 for positional encoding
        self.v_proj = nn.Linear(feature_dim + 64, head_dim * num_heads)
        
        # Self-attention layer
        self.self_attn = nn.MultiheadAttention(feature_dim, num_heads, batch_first=True)
        
        # Positional encoding function (simplified)
        self.pos_enc = PositionalEncoding(64)

    def forward(self, F, F_tilde, pose_I, pose_tilde, K, K_tilde, H, W):
        """
        F: (B, H, W, C) - Features from reference view I
        F_tilde: (B, H, W, C) - Features from secondary view I_tilde
        pose_I, pose_tilde: (B, 3, 4) - Camera extrinsics [R|t]
        K, K_tilde: (B, 3, 3) - Camera intrinsics
        """
        B, H, W, C = F.shape
        device = F.device
        
        # 1. Compute epipolar lines and samples for each pixel in I
        # This is vectorized over B and (H, W)
        # u: (B, H*W, 2) pixel coordinates in I
        # u_tilde_samples: (B, H*W, L, 2) sampled coordinates in I_tilde
        # depths: (B, H*W, L) triangulated depths
        
        u_coords = get_pixel_coordinates(H, W, device) # (H, W, 2)
        u_coords = u_coords.unsqueeze(0).unsqueeze(0).expand(B, H*W, -1) # (B, H*W, 2)
        
        # Compute epipolar geometry
        # For each pixel u in I, compute the epipolar line in I_tilde
        # Then sample L points along this line
        u_tilde_samples, depths = compute_epipolar_samples(
            u_coords, pose_I, pose_tilde, K, K_tilde, 
            num_samples=self.num_samples
        )
        # u_tilde_samples: (B, H*W, L, 2)
        # depths: (B, H*W, L)
        
        # 2. Sample features from F_tilde at u_tilde_samples
        # F_tilde is (B, H, W, C). We need to gather features at (u_tilde_samples)
        # Using grid_sample or custom gather
        F_tilde_samples = sample_features_at_coords(F_tilde, u_tilde_samples) # (B, H*W, L, C)
        
        # 3. Augment with positional encoding of depths
        # depths: (B, H*W, L) -> (B, H*W, L, 64)
        depth_enc = self.pos_enc(depths) 
        s = torch.cat([F_tilde_samples, depth_enc], dim=-1) # (B, H*W, L, C+64)
        
        # 4. Project to Q, K, V
        # q: (B, H*W, C) -> (B, H*W, H*head_dim)
        q = self.q_proj(F.view(B, -1, C)) # Flatten spatial dims for projection
        
        # k, v: (B, H*W, L, C+64) -> (B, H*W, L, H*head_dim)
        k = self.k_proj(s)
        v = self.v_proj(s)
        
        # Reshape for multi-head attention
        # q: (B, H*W, num_heads, head_dim)
        q = q.view(B, -1, self.num_heads, self.head_dim).transpose(1, 2) # (B, num_heads, H*W, head_dim)
        # k, v: (B, num_heads, H*W, L, head_dim)
        k = k.view(B, -1, self.num_heads, self.num_samples, self.head_dim).transpose(1, 3) # (B, num_heads, H*W, L, head_dim) -> (B, num_heads, L, H*W, head_dim) ? 
        # Standard MHAT expects (B, num_heads, N, D). Here N is the sequence length.
        # For cross-attention, Q is from I (N=H*W), K,V are from epipolar samples (M=L).
        # We need to attend from each pixel in I to L samples.
        
        # Let's reshape correctly for F.cross_attention(q, k, v)
        # q: (B, num_heads, H*W, head_dim)
        # k: (B, num_heads, L, H*W, head_dim) -> This is wrong. 
        # k should be (B, num_heads, L, head_dim) if we attend to L keys for each query?
        # No, standard cross-attention: Q is (B, N, D), K is (B, M, D), V is (B, M, D).
        # Here N = H*W (pixels in I), M = L (samples per pixel).
        # But we have H*W independent queries, each with L keys.
        # So we treat (B, H*W) as the batch dimension for the attention layer?
        # Or we flatten B and H*W.
        
        # Flatten B and H*W
        q_flat = q.view(B * H * W, self.num_heads, self.head_dim) # (B*H*W, num_heads, head_dim)
        k_flat = k.view(B * H * W, self.num_heads, self.num_samples, self.head_dim) # (B*H*W, num_heads, L, head_dim)
        v_flat = v.view(B * H * W, self.num_heads, self.num_samples, self.head_dim)
        
        # Compute attention scores
        # attn_weights: (B*H*W, num_heads, L)
        attn_weights = F.softmax(torch.matmul(q_flat, k_flat.transpose(-2, -1)) / (self.head_dim ** 0.5), dim=-1)
        
        # Weighted sum
        # attn_output: (B*H*W, num_heads, head_dim)
        attn_output = torch.matmul(attn_weights, v_flat)
        
        # Reshape back
        # attn_output: (B*H*W, num_heads, head_dim) -> (B, H*W, num_heads * head_dim)
        attn_output = attn_output.view(B, H * W, self.num_heads * self.head_dim)
        
        # Skip connection and reshape to (B, H, W, C)
        F_flat = F.view(B, -1, C)
        F_updated_flat = F_flat + attn_output # (B, H*W, C)
        F_updated = F_updated_flat.view(B, H, W, C)
        
        # 5. Per-Image Self-Attention
        # Flatten spatial dims for self-attention
        F_self_input = F_updated.view(B, H * W, C)
        # Self-attention expects (N, B, C) or (B, N, C) depending on batch_first
        # Using batch_first=True
        F_self_out, _ = self.self_attn(F_self_input, F_self_input, F_self_input)
        F_final = F_updated + F_self_out # Skip connection
        F_final = F_final.view(B, H, W, C)
        
        return F_final

def compute_epipolar_samples(u_coords, pose_I, pose_tilde, K, K_tilde, num_samples=64):
    """
    u_coords: (B, N, 2) pixel coordinates in I
    Returns:
    u_tilde_samples: (B, N, L, 2)
    depths: (B, N, L)
    """
    B, N, _ = u_coords.shape
    L = num_samples
    
    # 1. Unproject u_coords to 3D rays in I's camera frame
    # r_I: (B, N, 3)
    r_I = unproject_pixel(u_coords, K) # (B, N, 3)
    
    # 2. Transform rays to world frame
    # R_I, t_I from pose_I
    R_I = pose_I[:, :, :3, :3] # (B, 3, 3)
    t_I = pose_I[:, :, :3, 3] # (B, 3)
    
    # Ray origin in world: O_I = -R_I^T * t_I
    O_I = -torch.bmm(R_I.transpose(1, 2), t_I.unsqueeze(2)).squeeze(2) # (B, 3)
    O_I = O_I.unsqueeze(1).expand(-1, N, -1) # (B, N, 3)
    
    # Ray direction in world: D_I = R_I^T * r_I
    D_I = torch.bmm(R_I.transpose(1, 2), r_I.transpose(1, 2)).transpose(1, 2) # (B, N, 3)
    
    # 3. Project rays onto I_tilde's image plane to get epipolar lines
    # We need to find the line in I_tilde that corresponds to the ray (O_I, D_I)
    # The epipolar line l_tilde in I_tilde is given by l_tilde = K_tilde * E_tilde * F * K_I^-1 * u ?
    # Simpler: Project the ray into I_tilde.
    # The ray is a line in 3D. We can sample points on this ray and project them to I_tilde.
    # However, the epipolar line is the projection of the entire ray.
    # We can compute the epipolar line coefficients in I_tilde.
    
    # Fundamental Matrix approach: l_tilde = F * u
    # But we have poses.
    # E_tilde = [t_tilde]_x * R_tilde * R_I^T
    # F_tilde = K_tilde^-T * E_tilde * K_I^-1
    
    # Let's use the direct projection method for sampling.
    # Sample points along the ray in I's frame, transform to world, then to I_tilde.
    # But the ray is infinite. We need a range of depths.
    # The depth d is the distance from O_I to the point X.
    # X = O_I + d * D_I (where D_I is unit vector? No, r_I is not unit).
    # Let's normalize D_I.
    D_I_norm = F.normalize(D_I, dim=2)
    
    # Sample depths d_l along the ray. 
    # The depth d_l is the distance from O_I.
    # We need to determine the range of d_l. 
    # In practice, we sample d_l uniformly in a range [d_min, d_max] or based on scene scale.
    # For the encoder, the depth is part of the feature, so we sample a range.
    # Let's assume a fixed range or relative range.
    # The source says "compute its distance to I's camera origin".
    
    # Generate L depth values
    # d_l: (B, N, L)
    # We can sample d_l uniformly in [0, 1] and scale by a scene-dependent factor, 
    # or use a fixed range. The source doesn't specify the range, but implies it covers the scene.
    # Let's assume d_l is sampled from a distribution or fixed range.
    # For implementation, we might use a learnable or fixed set of depths.
    # However, the epipolar line is defined by the geometry, not the depth range.
    # The depth is computed by triangulation.
    
    # Correct approach:
    # 1. Define the epipolar line in I_tilde.
    # 2. Sample points on this line.
    # 3. Triangulate each point with u to get depth.
    
    # Epipolar line in I_tilde: l_tilde = K_tilde^-T * E_tilde * K_I^-1 * u_homogeneous
    # u_homogeneous: (B, N, 3)
    u_hom = torch.cat([u_coords, torch.ones(B, N, 1, device=u_coords.device)], dim=2)
    
    # Compute Fundamental Matrix from poses
    # E_tilde = [t_tilde]_x * R_tilde * R_I^T
    R_tilde = pose_tilde[:, :, :3, :3]
    t_tilde = pose_tilde[:, :, :3, 3]
    
    # Skew-symmetric matrix for t_tilde
    def skew(v):
        x, y, z = v.unbind(dim=-1)
        return torch.stack([
            torch.zeros_like(x), -z, y,
            z, torch.zeros_like(x), -x,
            -y, x, torch.zeros_like(x)
        ], dim=-1).view(B, 3, 3)
        
    E_tilde = torch.bmm(skew(t_tilde), torch.bmm(R_tilde, R_I.transpose(1, 2)))
    
    # F_tilde = K_tilde^-T * E_tilde * K_I^-1
    K_I_inv = torch.inverse(K)
    K_tilde_inv_T = torch.inverse(K_tilde).transpose(1, 2)
    F_tilde = torch.bmm(torch.bmm(K_tilde_inv_T, E_tilde), K_I_inv)
    
    # l_tilde = F_tilde * u_hom
    l_tilde = torch.bmm(F_tilde, u_hom.transpose(1, 2)).transpose(1, 2) # (B, N, 3)
    # l_tilde = (a, b, c) such that a*x + b*y + c = 0
    
    # Sample L points on the line l_tilde in I_tilde
    # We can sample two points on the line and interpolate, or sample directly.
    # A robust way: sample u_tilde_l in I_tilde image coordinates.
    # The line extends across the image. We can find the intersection with the image boundaries.
    # For simplicity, we sample L points uniformly along the line segment within the image.
    
    # Get image boundaries
    h, w = H, W
    # Points on the line: x = - (b*y + c) / a if a != 0
    # Or parametric form.
    
    # Let's sample L points by generating random y and solving for x, or vice versa.
    # To ensure coverage, we can sample y in [0, H] and compute x.
    # But we need to clip to image bounds.
    
    # Alternative: Sample L points on the line in homogeneous coordinates.
    # The line is defined by l_tilde.
    # We can sample points p_l = l_tilde x q_l, where q_l are points at infinity?
    # Simpler: Sample L points along the line in the image plane.
    
    # Let's assume we have a function to sample L points on a line within image bounds.
    u_tilde_samples = sample_line_points(l_tilde, H, W, L) # (B, N, L, 2)
    
    # 4. Triangulate u and u_tilde_l to get depth
    # For each u_tilde_l, compute the ray in I_tilde
    r_tilde_l = unproject_pixel(u_tilde_samples, K_tilde) # (B, N, L, 3)
    
    # Transform r_tilde_l to world frame
    R_tilde_inv = R_tilde.transpose(1, 2)
    t_tilde = pose_tilde[:, :, :3, 3]
    
    # Ray origin in world: O_tilde = -R_tilde^T * t_tilde
    O_tilde = -torch.bmm(R_tilde_inv, t_tilde.unsqueeze(2)).squeeze(2) # (B, 3)
    O_tilde = O_tilde.unsqueeze(1).unsqueeze(2).expand(-1, N, L, -1) # (B, N, L, 3)
    
    # Ray direction in world: D_tilde = R_tilde^T * r_tilde_l
    D_tilde = torch.bmm(R_tilde_inv, r_tilde_l.transpose(2, 3)).transpose(2, 3) # (B, N, L, 3)
    D_tilde_norm = F.normalize(D_tilde, dim=3)
    
    # Triangulate O_I, D_I_norm and O_tilde, D_tilde_norm
    # X = O_I + d_I * D_I_norm = O_tilde + d_tilde * D_tilde_norm
    # Solve for d_I (depth in I)
    # This is a linear least squares problem or closed form.
    # d_I = ( (O_tilde - O_I) x D_tilde_norm ) . D_I_norm x D_tilde_norm ) / | D_I_norm x D_tilde_norm |^2
    # Or use SVD.
    
    # Let's use the closed form for minimal triangulation error.
    # A = [D_I_norm; -D_tilde_norm]
    # b = O_tilde - O_I
    # Solve A * [d_I; -d_tilde] = b ? No.
    # X = O_I + d_I * D_I = O_tilde + d_tilde * D_tilde
    # d_I * D_I - d_tilde * D_tilde = O_tilde - O_I
    
    # We want d_I.
    # Cross product with D_tilde:
    # d_I * (D_I x D_tilde) = (O_tilde - O_I) x D_tilde
    # d_I = ((O_tilde - O_I) x D_tilde) . (D_I x D_tilde) / |D_I x D_tilde|^2
    
    diff = O_tilde - O_I # (B, N, L, 3)
    cross_prod = torch.cross(D_I_norm.unsqueeze(2), D_tilde_norm, dim=3) # (B, N, L, 3)
    cross_prod_sq = torch.sum(cross_prod ** 2, dim=3, keepdim=True) # (B, N, L, 1)
    
    # Avoid division by zero
    cross_prod_sq = torch.clamp(cross_prod_sq, min=1e-8)
    
    numerator = torch.sum(diff * cross_prod, dim=3, keepdim=True) # (B, N, L, 3) . (B, N, L, 3) -> (B, N, L)
    depths = numerator / cross_prod_sq.squeeze(3) # (B, N, L)
    
    return u_tilde_samples, depths

def sample_features_at_coords(F, coords):
    """
    F: (B, H, W, C)
    coords: (B, N, L, 2) normalized coordinates [-1, 1]
    Returns: (B, N, L, C)
    """
    # Normalize coords to [-1, 1]
    # coords are in [0, W] and [0, H]
    # grid_sample expects (B, C, H, W) and grid (B, H_out, W_out, 2)
    # We need to reshape F to (B, C, H, W)
    F_perm = F.permute(0, 3, 1, 2) # (B, C, H, W)
    
    # Normalize coords
    # x in [0, W] -> [-1, 1]
    # y in [0, H] -> [-1, 1]
    # grid_sample expects (x, y) in [-1, 1]
    x = coords[..., 0] / (W - 1) * 2 - 1
    y = coords[..., 1] / (H - 1) * 2 - 1
    grid = torch.stack([x, y], dim=-1) # (B, N, L, 2)
    
    # Reshape for grid_sample
    # grid_sample input grid: (B, H_out, W_out, 2)
    # We want to sample L points for each of N pixels.
    # This is not a standard grid_sample call.
    # We need to use a custom gather or interpolate.
    # Since L is small, we can loop or use advanced indexing.
    
    # Advanced indexing
    # coords are continuous. We need to interpolate.
    # For simplicity, assume nearest neighbor or bilinear.
    # Using grid_sample with a dummy grid? No.
    
    # Let's use F.grid_sample with a grid of shape (B, N*L, 1, 2)?
    # No, grid_sample expects a 2D grid.
    
    # We can reshape N and L into the batch dimension?
    # F: (B, C, H, W)
    # grid: (B*N*L, 1, 1, 2)
    # This is inefficient.
    
    # Better: Use torch.nn.functional.grid_sample with a grid of shape (B, N, L, 2) is not supported.
    # We must reshape.
    
    # Reshape F to (B*N, C, H, W)
    # Reshape grid to (B*N, L, 1, 2)
    # Then grid_sample gives (B*N, C, L, 1)
    # Then reshape to (B, N, L, C)
    
    B, N, L, _ = coords.shape
    F_reshaped = F_perm.view(B * N, C, H, W)
    grid_reshaped = grid.view(B * N, L, 1, 2)
    
    # grid_sample
    sampled = F.grid_sample(F_reshaped, grid_reshaped, align_corners=True) # (B*N, C, L, 1)
    
    # Reshape back
    sampled = sampled.view(B, N, C, L, 1) # (B, N, C, L, 1)
    sampled = sampled.permute(0, 1, 3, 2, 4).squeeze(-1) # (B, N, L, C)
    
    return sampled
```

### Parameters
*   **Number of samples ($L$):** Typically 64.
*   **Feature dimension ($C$):** Depends on the backbone (e.g., 256, 512).
*   **Head dimension ($C_{head}$):** Typically 64.
*   **Positional Encoding:** Standard sine/cosine or learned.

### Edge Cases and Pitfalls
*   **Missing Correspondences:** Pixels in $I$ that have no match in $\tilde{I}$ (occlusions, boundaries) will have no valid epipolar samples. The per-image self-attention layer propagates depth estimates from neighboring pixels to these regions.
*   **Triangulation Instability:** If the rays from $I$ and $\tilde{I}$ are nearly parallel, the triangulation depth $\tilde{d}_{\tilde{\mathbf{u}}_l}$ becomes noisy. This is mitigated by the attention mechanism, which learns to down-weight unreliable samples.
*   **Scale Ambiguity:** Without the positional encoding of depth, the encoder cannot resolve the arbitrary scale factor $s_i$. The depth augmentation is critical.

## Subtopics

### Epipolar Transformer (for Scale Ambiguity)
A multi-view encoder component that resolves scale ambiguity by finding cross-view pixel correspondences and associating them with positionally encoded depth values, allowing the inference of the per-scene scale factor $s_i$. It compares features belonging to a ray's corresponding pixel on one view with features sampled along the epipolar line on the other view. Epipolar samples are augmented with their positionally-encoded depths along the ray. Recorded depths are later used for depth prediction. [S1]

### Multi-View Epipolar Encoder
Extends the epipolar encoder from pairs to an arbitrary number of reference views by taking epipolar samples from any number of images and using their union. For a given pixel coordinate, epipolar samples are taken from any number of images. The union of these samples is used in place of a single epipolar line's samples. A learnable per-image embedding is added to each sampled feature to allow the epipolar transformer to distinguish between samples taken from different views. [S1]

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L \cdot C^2)$ per pixel, where $N$ is the number of pixels, $L$ is the number of epipolar samples, and $C$ is the feature dimension. The self-attention layer adds $O(N^2 \cdot C)$ if applied globally, but is typically applied per-image with linear complexity or reduced heads.
*   **Memory Complexity:** $O(N \cdot L \cdot C)$ to store the epipolar samples and intermediate features.
*   **Performance:** In 3-view settings, PSNR improves from 26.09 to 28.31 on the Real Estate 10k dataset, with SSIM improving from 0.863 to 0.908. [S1]
*   **Strengths:** Resolves scale ambiguity without explicit depth supervision; leverages geometric constraints for robust feature aggregation.
*   **Limitations:** Requires accurate camera poses; performance degrades if epipolar geometry is incorrect; self-attention may blur details in textureless regions.

## Background
The Epipolar Encoder was introduced in "PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction" (CVPR 2024). It addresses the limitation of diffusion models that rescale 3D scenes according to heuristics on depth statistics. By using an epipolar transformer, the method infers the scene scale directly from cross-view correspondences. It builds on the concept of epipolar attention [16] and is used in conjunction with 3D Gaussian Splatting [19] for novel view synthesis.

## Sources
[S1] PixelSplat: 3D Gaussian Splats from Image Pairs for Scalable Generalizable 3D Reconstruction, https://doi.org/10.1109/cvpr52733.2024.01840, fetched 2026-10-26
