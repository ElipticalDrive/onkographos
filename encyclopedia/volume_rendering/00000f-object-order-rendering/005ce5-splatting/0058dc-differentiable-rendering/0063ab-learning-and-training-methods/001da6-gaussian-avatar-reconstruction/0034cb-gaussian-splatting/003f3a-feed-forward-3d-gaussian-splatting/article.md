# Feed-forward 3D Gaussian Splatting

Also called: Feed-forward Reconstruction Model

## Overview
Feed-forward 3D Gaussian Splatting explicitly represents a 3D scene as a collection of anisotropic Gaussian distributions to enable real-time novel view synthesis via splatting [S1]. It predicts Gaussian primitives directly from $N$ input images using a multi-view Transformer and linear projection, bypassing iterative optimization [S2].

## How it works
**Inputs:** $N$ RGB images of resolution $H \times W$ with known camera intrinsics and extrinsics [S2].
**Outputs:** A set of $K$ 3D Gaussian primitives, where each primitive $k$ is defined by parameters $\mathbf{G}_k$ and feature vector $\mathbf{F}_k$ [S2].

**Data Structures:**
1.  **Input Images:** $N$ tensors of shape $(H, W, 3)$ [S2].
2.  **Ray Embeddings:** Plücker ray embeddings computed per pixel [S2].
3.  **Patch Features:** Extracted via a shallow CNN, shape $(N, \frac{HW}{p^2}, C)$ [S2].
4.  **Token Sequence:** Flattened patch features, shape $(N \times \frac{HW}{p^2}, C)$ [S2].
5.  **Latent Features:** Output of the multi-view Transformer, shape $(N \times \frac{HW}{p^2}, C_{\text{latent}})$ [S2].
6.  **Gaussian Primitives:** Each includes center $\mu \in \mathbb{R}^3$ [S1], covariance $\Sigma \in \mathbb{R}^{3 \times 3}$ parameterized by orthogonal rotation $R$ and diagonal scale $S$ such that $\Sigma = RSS^T R^T$ [S1], spherical harmonic coefficients $c_{\text{SH}} \in \mathbb{R}^{3 \times (k+1)}$ [S1], opacity $\alpha \in \mathbb{R}^1$ [S1], and additional feature vector $\mathbf{F}_k$ [S2].

**Steps:**
1.  **Feature Extraction:** Concatenate RGB values with Plücker ray embeddings per pixel [S2].
2.  **Patch Encoding:** Group features into non-overlapping $p \times p$ patches [S2]. Encode with a shallow CNN to obtain patch features of channel dimension $C$ [S2].
3.  **Tokenization:** Flatten patch features from all $N$ views into a sequence $\{\mathbf{X}\}$ of length $N \times \frac{HW}{p^2}$ [S2].
4.  **Transformer Processing:** Process $\{\mathbf{X}\}$ with a multi-view Transformer to produce latent features $\{\mathbf{Z}\}$: $\{\mathbf{Z}\} = \text{Transformer}_{\text{MV}}(\{\mathbf{X}\})$ [S2].
5.  **Primitive Prediction:** Apply a linear projection to each latent token to produce $\{\mathbf{G}_k, \mathbf{F}_k\} = \text{Linear}(\{\mathbf{Z}\})$ [S2].
6.  **Rendering:** Project each 3D Gaussian onto the 2D image plane for a target viewpoint: $\mu_{2D} = P_W \mu$, $\Sigma_{2D} = J_W \Sigma_W^T J^T$ [S1]. Compute pixel color via front-to-back alpha compositing: $C(\mathbf{p}) = \sum_{i=1}^K \mathbf{c}_i \alpha_i \mathcal{G}_i^{\text{2D}}(\mathbf{p}) \prod_{j=1}^{i-1} (1 - \alpha_j \mathcal{G}_j^{\text{2D}}(\mathbf{p}))$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_plucker(origins, directions):
    """Compute Plücker coordinates L = o x d for each pixel."""
    return np.cross(origins, directions)

def compute_ray_data(camera, H, W):
    """Compute ray origins and directions from camera matrices."""
    K = camera['intrinsics']
    R, t = camera['extrinsics'][:3, :3], camera['extrinsics'][:3, 3]
    origin = -R.T @ t
    inv_K = np.linalg.inv(K)
    y, x = np.meshgrid(np.arange(H), np.arange(W), indexing='ij')
    hom = np.stack([x, y, np.ones_like(x)], axis=-1).astype(np.float32)
    dirs = (inv_K @ hom.reshape(-1, 3).T).T.reshape(H, W, 3)
    dirs = (R.T @ dirs.reshape(-1, 3).T).T.reshape(H, W, 3)
    return origin, dirs

def shallow_cnn(patches, channels=64):
    """Apply a shallow CNN to patch features."""
    # Simulates a 3x3 convolution followed by a linear projection to `channels`
    # In a full implementation, this would use convolutional kernels.
    return patches @ np.random.randn(patches.shape[1], channels)

def multi_view_transformer(tokens, dim=512, num_heads=8, num_layers=12):
    """Process tokens with a multi-view Transformer."""
    # Add positional encodings
    pos_enc = np.random.randn(tokens.shape[0], dim)
    x = tokens + pos_enc
    # Apply multi-head self-attention layers
    for _ in range(num_layers):
        # Simplified attention block
        attn = (x @ np.random.randn(dim, dim))
        x = x + attn
    return x

def reconstruct_covariance(cov_flat):
    """Reconstruct symmetric 3x3 covariance matrix from 6 upper-triangle values."""
    return np.array([
        [cov_flat[0], cov_flat[1], cov_flat[2]],
        [cov_flat[1], cov_flat[3], cov_flat[4]],
        [cov_flat[2], cov_flat[4], cov_flat[5]]
    ])

def feed_forward_gaussian_splatting(images, cameras, patch_size=16):
    """
    Predicts 3D Gaussian primitives from N RGB images.
    
    Args:
        images: List of N numpy arrays, shape (H, W, 3), dtype float32, normalized [0, 1].
        cameras: List of N camera objects with 'extrinsics' (4x4) and 'intrinsics' (3x3).
        patch_size: Integer, size of non-overlapping patches.
        
    Returns:
        gaussians: List of dictionaries, each containing 'mu', 'cov', 'alpha', 'sh_coeffs', 'features'.
    """
    N = len(images)
    H, W, _ = images[0].shape
    num_patches_h = H // patch_size
    num_patches_w = W // patch_size
    num_patches = num_patches_h * num_patches_w

    # Step 1: Feature Extraction
    pixel_features = []
    for i in range(N):
        origins, directions = compute_ray_data(cameras[i], H, W)
        plucker = compute_plucker(origins, directions)
        plucker_expanded = np.repeat(plucker[:, :, np.newaxis, :], 3, axis=2).reshape(H, W, 6)
        combined = np.concatenate([images[i], plucker_expanded], axis=-1)
        pixel_features.append(combined)
        
    # Step 2: Patch Encoding
    patches = []
    for i in range(N):
        feat = pixel_features[i]
        num_patches_h = H // patch_size
        num_patches_w = W // patch_size
        reshaped = feat.reshape(num_patches_h, patch_size, num_patches_w, patch_size, 9)
        patch_data = reshaped.reshape(num_patches_h * num_patches_w, patch_size * patch_size * 9)
        patches.append(patch_data)
        
    all_patches = np.stack(patches, axis=0)
    batch_size = N * num_patches
    flat_patches = all_patches.reshape(batch_size, -1)
    cnn_output = shallow_cnn(flat_patches, channels=64)
    patch_features = cnn_output.reshape(N, num_patches, 64)
    
    # Step 3: Tokenization
    tokens = patch_features.reshape(-1, 64)
    
    # Step 4: Transformer Processing
    latents = multi_view_transformer(tokens, dim=512, num_heads=8, num_layers=12)
    
    # Step 5: Primitive Prediction
    D_out = 3 + 6 + 1 + 27 + 32
    linear_layer = np.random.randn(latents.shape[1], D_out)
    predictions = latents @ linear_layer
    
    # Step 6: Parse predictions into Gaussian primitives
    gaussians = []
    for k in range(predictions.shape[0]):
        pred = predictions[k]
        mu = pred[0:3]
        cov_flat = pred[3:9]
        alpha = pred[9]
        sh_coeffs = pred[10:37]
        features = pred[37:]
        cov = reconstruct_covariance(cov_flat)
        gaussians.append({
            'mu': mu,
            'cov': cov,
            'alpha': alpha,
            'sh_coeffs': sh_coeffs,
            'features': features
        })
        
    return gaussians
```

**Edge Cases and Pitfalls:**
*   **Covariance PSD:** The covariance matrix $\Sigma$ must remain positive semi-definite. The method ensures this by parameterizing $\Sigma = RSS^T R^T$, where $R$ is an orthogonal rotation matrix and $S$ is a diagonal scale matrix [S1].
*   **Patch Size:** The patch size $p$ must divide $H$ and $W$ evenly to form non-overlapping patches [S2].
*   **Plücker Embeddings:** Computed from ray origins and directions derived from known camera intrinsics and extrinsics [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Enables real-time novel view synthesis after a single forward pass [S2]. Quality is competitive with NeRF [S1].
*   **Strengths:** Fast reconstruction without per-scene training [S2].
*   **Limitations:** Not given in the sources.

## Background
Feed-forward 3D Gaussian Splatting builds on [Gaussian Splatting](946-gaussian-splatting.md) [S1]. The specific feed-forward reconstruction model is based on the architecture from GS-LRM [S2].

## Sources
[S1] DisC-GS: Discontinuity-aware Gaussian Splatting, https://arxiv.org/abs/2405.15196v2, fetched 2026-10-26
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
