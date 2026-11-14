# ViT-NeRF

Also called: Global-Local Feature NeRF, Hybrid ViT-CNN NeRF, Vision Transformer for NeRF (ViT-NeRF)

## Overview
ViT-NeRF is a view synthesis method that combines a Vision Transformer (ViT) to encode global shape information and a 2D CNN to extract local appearance features, conditioning volume rendering on hybrid per-pixel features [S1]. Unlike CodeNeRF and DISN, this method does not require a canonical coordinate system to utilize global features [S1]. Compared to SRN and PixelNeRF, it produces sharper edges and clearer structures, avoiding blurriness on invisible parts like chair legs [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A single input image $I_s \in \mathbb{R}^{H \times W \times 3}$, where $H$ and $W$ are height and width [S1].
*   **Output:** A rendered 2D image for a target viewpoint [S1].

**Data Structures**
*   **Image Tokens:** Sequence of flattened 2D patches $P \in \mathbb{R}^{N \times P^2 \times 3}$, where $N = \frac{HW}{P^2}$ is the number of patches and $P$ is the patch size [S1].
*   **Latent Features:** Set of latent features $f$ extracted by the ViT encoder [S1].
*   **Global Feature Maps:** Multi-level global feature maps $W^0_G, ..., W^J_G$, where $W^j_G \in \mathbb{R}^{H_j \times W_j \times D_j}$ [S1].
*   **Local Feature Map:** Local feature map $W_L \in \mathbb{R}^{\frac{H}{2} \times \frac{W}{2} \times D_L}$ [S1].
*   **Hybrid Feature Map:** Fused feature map $W$ [S1].
*   **Per-pixel Feature:** Sampled feature $W(\pi(x))$ at pixel coordinate $\pi(x)$ [S1].

**Parameters**
*   **Patch Size ($P$):** Not given in the sources.
*   **Number of Frequency Bases ($M$):** 10 [S1].
*   **Learning Rate:** $10^{-4}$ for the NeRF MLP; $10^{-5}$ for the ViT and CNN modules [S1].
*   **Batch Size:** 8 instances [S1].
*   **Rays per Instance:** 512 rays [S1].
*   **Training Iterations:** Converges at 500K iterations [S1].
*   **Warm-up Steps:** 10k steps [S1].
*   **ViT Initialization:** Pretrained weights from reference [55] (DINO), fine-tuned [S1].
*   **Local CNN Architecture:** $G_L$ consists of three ResBlocks [S1].

**Steps**

1.  **Global Feature Extraction:**
    1.  Reshape input image $I_s$ into a sequence of flattened 2D patches $P \in \mathbb{R}^{N \times P^2 \times 3}$ [S1].
    2.  Linearly project each patch to an image token $P_l$ [S1].
    3.  Input image tokens and learnable positional embeddings $e$ into the Transformer Encoder to extract global latent features $f$ [S1].
    4.  Drop the class token from $f$. The class token has no physical meaning when unflattened into spatial feature maps [S1]. Let $O: \mathbb{R}^{(N+1) \times D} \rightarrow \mathbb{R}^{N \times D}$ be the operation of dropping the class token [S1].
    5.  Unflatten the remaining tokens using operation $U: \mathbb{R}^{N \times D} \rightarrow \mathbb{R}^{\frac{H}{P} \times \frac{W}{P} \times D}$ to obtain feature patches $P_f = \{P^0_f, ..., P^J_f\}$, where $P^j_f \in \mathbb{R}^{\frac{H}{P} \times \frac{W}{P} \times D}$ [S1].
    6.  Generate multi-level global feature maps $W^j_G$ using convolutional decoders $D^j$ for each level $j \in \{0, ..., J\}$ [S1]:
        $$W^j_G = (D^j \circ U \circ O)(f^j)$$
        Each decoder $D^j: \mathbb{R}^{\frac{H}{P} \times \frac{W}{P} \times D} \rightarrow \mathbb{R}^{H_j \times W_j \times D_j}$ performs: (i) $1 \times 1$ convolution, (ii) resampling via strided or transposed convolution to size $H_j \times W_j$, and (iii) $3 \times 3$ convolution to $D_j$ channels [S1].

2.  **Local Feature Extraction:**
    1.  Pass input image $I_s$ through a 2D CNN module $G_L$ to extract local features $W_L$ [S1]:
        $$W_L = G_L(I_s)$$
        where $G_L: \mathbb{R}^{H \times W \times 3} \rightarrow \mathbb{R}^{\frac{H}{2} \times \frac{W}{2} \times D_L}$ [S1].

3.  **Feature Fusion:**
    1.  Fuse global maps $W^0_G, ..., W^J_G$ and local map $W_L$ using a convolutional layer $G$ to produce the hybrid feature map $W$ [S1]:
        $$W = G(W^0_G, W^1_G, ..., W^J_G; W_L)$$
        [S1]

4.  **Volume Rendering:**
    1.  For a target viewpoint, define a ray $r(t) = o + td$, where $o$ is the origin, $d$ is the direction, and $t \in [t_{near}, t_{far}]$ [S1].
    2.  Sample equally distributed points $x_c$ along the ray [S1].
    3.  Encode 3D location $x_c$ with positional encoding $\gamma(x_c)$ using $M=10$ frequency bases [S1]:
        $$\gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), ..., \sin(2^{M-1} \pi p), \cos(2^{M-1} \pi p))$$
        [S1]
    4.  Sample the per-pixel feature $W(\pi(x))$ corresponding to the pixel coordinate $\pi(x)$ of $x_c$ [S1].
    5.  Input $\gamma(x_c)$, viewing direction $d_c$, and $W(\pi(x))$ into an MLP to output density $\sigma$ and color $c$ [S1]:
        $$(\sigma, c) = \text{MLP}(\gamma(x_c); d_c; W(\pi(x)))$$
        [S1]
    6.  Render the image using standard volume rendering [S1]:
        $$\hat{C}(r) = \int_{t_n}^{t_f} T(t)\sigma(t)c(t)dt$$
        where $T(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s))ds\right)$ is the accumulated transmittance [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ViTNeRF:
    def __init__(self, H, W, patch_size, num_freq_bases=10):
        """
        Initialize ViT-NeRF components.
        """
        self.H = H
        self.W = W
        self.patch_size = patch_size
        self.M = num_freq_bases
        self.N = (H * W) // (patch_size ** 2)
        
        # Global Feature Branch: ViT Encoder
        # Note: Specific ViT architecture details (layers, heads) are not given in sources [S1]
        # Assuming a standard ViT encoder that outputs latent features f
        self.vit_encoder = ViTEncoder() 
        
        # Global Feature Branch: Convolutional Decoder
        # Generates multi-level feature maps W^j_G
        # Note: Specific decoder layers (D^j) are not fully detailed in sources [S1]
        self.conv_decoder = ConvDecoder()
        
        # Local Feature Branch: 2D CNN
        # Consists of three ResBlocks [S1]
        self.local_cnn = LocalCNN()
        
        # Fusion Layer
        # Convolutional layer G to fuse global and local features [S1]
        self.fusion_layer = nn.Conv2d(in_channels=..., out_channels=..., kernel_size=...) # Dimensions not given in sources [S1]
        
        # Rendering MLP
        # Takes positional encoding, viewing direction, and hybrid feature [S1]
        self.mlp = nn.Sequential(
            nn.Linear(..., ...), # Input dim: 3*M (pos enc) + 3 (view dir) + W_channels [S1]
            nn.ReLU(),
            # ... additional layers not given in sources [S1]
            nn.Linear(..., 4) # Output: sigma (1) + color (3) [S1]
        )

    def positional_encoding(self, p):
        """
        Encode 3D position p with M frequency bases [S1].
        """
        # gamma(p) = (sin(2^0 pi p), cos(2^0 pi p), ..., sin(2^(M-1) pi p), cos(2^(M-1) pi p))
        # p is shape (..., 3)
        freq_bands = 2.0 ** torch.arange(self.M, device=p.device).float() * torch.pi
        # Expand dimensions for broadcasting
        freq_bands = freq_bands.view(1, 1, -1)
        p_expanded = p.unsqueeze(-1) * freq_bands # Shape: (..., 3, M)
        
        pe = torch.cat([torch.sin(p_expanded), torch.cos(p_expanded)], dim=-1)
        # Flatten the last two dimensions to get (..., 2*M)
        return pe.view(*p.shape[:-1], -1)

    def forward(self, input_image, ray_origins, ray_directions, near, far):
        """
        Render a view given ray parameters [S1].
        """
        # 1. Global Features
        # Flatten and project input image to patches [S1]
        # Note: Exact patch projection layer not specified, assuming standard ViT input processing
        patches = self.extract_patches(input_image) # Shape: (B, N, P^2 * 3)
        image_tokens = self.vit_encoder.patch_embedding(patches) # Shape: (B, N, D)
        
        # Add positional embeddings [S1]
        pos_embed = self.vit_encoder.pos_embed
        x = image_tokens + pos_embed
        
        # ViT Encoder to get latent features f [S1]
        latent_features = self.vit_encoder.blocks(x)
        
        # Drop class token [S1]
        # Assuming class token is at index 0
        latent_features_no_cls = latent_features[:, 1:, :] # Shape: (B, N, D)
        
        # Unflatten to spatial feature patches [S1]
        # H/P x W/P spatial dimensions
        h_grid = self.H // self.patch_size
        w_grid = self.W // self.patch_size
        feature_patches = latent_features_no_cls.view(latent_features_no_cls.size(0), h_grid, w_grid, -1)
        
        # Generate multi-level global feature maps W^j_G [S1]
        # Note: Specific multi-level generation logic (strided/transpose convs) is abstracted here
        global_maps = self.conv_decoder(feature_patches) # List of maps W^0_G ... W^J_G
        
        # 2. Local Features
        # Extract local features W_L using 2D CNN [S1]
        local_map = self.local_cnn(input_image) # Shape: (B, H/2, W/2, D_L)
        
        # 3. Fusion
        # Fuse global maps and local map using convolutional layer G [S1]
        # Note: Concatenation of multi-level global maps and local map before fusion is implied
        hybrid_feature = self.fusion_layer(torch.cat(global_maps + [local_map], dim=1))
        
        # 4. Volume Rendering
        # Sample points along rays [S1]
        t_vals = torch.linspace(near, far, 128, device=input_image.device) # Example sample count
        xyz = ray_origins.unsqueeze(1) + ray_directions.unsqueeze(1) * t_vals.unsqueeze(0).unsqueeze(-1)
        
        # Encode 3D locations [S1]
        pos_enc = self.positional_encoding(xyz)
        
        # Sample per-pixel features W(pi(x)) [S1]
        # Note: Requires mapping 3D points to 2D pixel coordinates and sampling from hybrid_feature
        # This step depends on the specific projection from 3D to 2D for the target view
        # Assuming a function `sample_hybrid_features` exists that handles this
        pixel_features = sample_hybrid_features(hybrid_feature, xyz, ray_directions)
        
        # View direction encoding (optional, often concatenated) [S1]
        # The MLP input includes viewing direction d_c [S1]
        
        # MLP to get sigma and color [S1]
        mlp_input = torch.cat([pos_enc, pixel_features, ray_directions], dim=-1)
        outputs = self.mlp(mlp_input)
        sigma = outputs[..., 0:1]
        color = outputs[..., 1:4]
        
        # Standard volume rendering integral [S1]
        weights = compute_weights(t_vals, sigma)
        rendered_color = torch.sum(weights.unsqueeze(-1) * color, dim=-2)
        
        return rendered_color

    def extract_patches(self, image):
        # Reshape image into flattened 2D patches [S1]
        # B, H, W, 3 -> B, N, P^2 * 3
        # Implementation details for patch extraction not explicitly given in sources [S1]
        pass
```

**Edge Cases and Pitfalls**
*   Using only global ViT features compromises rendering quality for views close to the input view, resulting in inconsistent color and appearance [S1]. This is solved by adding the local CNN branch [S1].
*   The class token in ViT has no physical meaning when unflattened into spatial feature maps and must be dropped before decoding [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Trained on 16 NVIDIA A100 GPUs, converging at 500K iterations [S1]. Batch size is 8 instances with 512 rays per instance [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Produces sharper edges and clearer structures compared to SRN and PixelNeRF, avoiding blurriness on invisible parts like chair legs [S1]. Does not require a canonical coordinate system to utilize global features, unlike CodeNeRF and DISN [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When high-quality view synthesis is needed from a single input image, particularly where global shape consistency and local appearance fidelity are both required [S1].
*   **When not:** Not given in the sources.

## Background
ViT-NeRF was introduced in the paper "Vision Transformer for NeRF-Based View Synthesis from a Single Input Image" [S1]. It builds on the success of Vision Transformers in learning long-range dependencies between features [S1]. The method combines the advantages of 2D-based methods (conditioning on local image features) and 1D-based methods (encoding global information) [S1].

## Sources
[S1] Vision Transformer for NeRF-Based View Synthesis from a Single Input Image, https://doi.org/10.1109/wacv56688.2023.00087, fetched 2026-10-27
