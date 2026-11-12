# Feed-forward 3D Gaussian Splatting

Also called: Feed-forward Reconstruction Model

## Overview
Feed-forward 3D Gaussian Splatting is a method that predicts a complete 3D Gaussian representation of a scene directly from a sparse set of input images, bypassing the iterative optimization required by standard [Gaussian Splatting](946-gaussian-splatting.md). It solves the problem of slow reconstruction by using a multi-view Transformer to map image features to Gaussian primitives in a single forward pass. It differs from iterative methods by trading per-scene training time for inference latency, and from other feed-forward approaches by explicitly predicting the full set of Gaussian attributes (position, covariance, opacity, color) via a linear projection of transformer latents.

## How it works
**Inputs:** $N$ RGB images of resolution $H \times W$ with known camera intrinsics and extrinsics.
**Outputs:** A set of $K$ 3D Gaussian primitives, where each primitive $k$ is defined by parameters $\mathbf{G}_k$ and feature vector $\mathbf{F}_k$.

**Data Structures:**
1.  **Input Images:** $N$ tensors of shape $(H, W, 3)$.
2.  **Ray Embeddings:** Plücker ray embeddings [S2] computed for each pixel based on camera extrinsics/intrinsics.
3.  **Patch Features:** Extracted via a shallow CNN. Shape: $(N, \frac{HW}{p^2}, C)$, where $p$ is the patch size and $C$ is the channel dimension.
4.  **Token Sequence:** Flattened patch features. Shape: $(N \times \frac{HW}{p^2}, C)$.
5.  **Latent Features:** Output of the multi-view Transformer. Shape: $(N \times \frac{HW}{p^2}, C_{latent})$.
6.  **Gaussian Primitives:** Predicted parameters. Each primitive includes:
    *   Center $\mu \in \mathbb{R}^3$ [S1].
    *   Covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$, parameterized by rotation $R \in \mathbb{R}^{3 \times 3}$ (orthogonal) and scale $S \in \mathbb{R}^{3 \times 3}$ (diagonal) such that $\Sigma = RSS^T R^T$ [S1].
    *   Spherical Harmonic coefficients $c_{SH} \in \mathbb{R}^{3 \times (k+1)}$ for view-dependent color [S1].
    *   Opacity $\alpha \in \mathbb{R}^1$ [S1].
    *   Additional feature vector $\mathbf{F}_k$ for downstream refinement [S2].

**Steps:**
1.  **Feature Extraction:** For each of the $N$ input images, extract per-pixel features by concatenating the RGB values with Plücker ray embeddings [S2].
2.  **Patch Encoding:** Group the per-pixel features into non-overlapping $p \times p$ image patches [S2]. Encode each patch using a shallow CNN to obtain patch features of channel dimension $C$ [S2].
3.  **Tokenization:** Flatten the patch features from all $N$ views into a single sequence of tokens $\{\mathbf{X}\}$ with length $N \times \frac{HW}{p^2}$ [S2].
4.  **Transformer Processing:** Process the token sequence $\{\mathbf{X}\}$ using a multi-view Transformer to produce latent features $\{\mathbf{Z}\}$ [S2]:
    $$ \{\mathbf{Z}\} = \text{Transformer}_{MV}(\{\mathbf{X}\}) $$
5.  **Primitive Prediction:** Apply a linear projection to each latent token in $\{\mathbf{Z}\}$ to produce the initial set of Gaussian primitives $\{\mathbf{G}_k\}$ and associated feature vectors $\{\mathbf{F}_k\}$ [S2]:
    $$ \{\mathbf{G}_k, \mathbf{F}_k\} = \text{Linear}(\{\mathbf{Z}\}) $$
6.  **Rendering (Optional/Downstream):** To render an image for a target viewpoint, project each 3D Gaussian onto the 2D image plane. The center $\mu_{2D}$ and covariance $\Sigma_{2D}$ of the projected Gaussian are computed as [S1]:
    $$ \mu_{2D} = P_W \mu, \quad \Sigma_{2D} = J_W \Sigma_W^T J^T $$
    where $W$ is the viewing transformation matrix, $P$ is the projective transformation matrix, and $J$ is the Jacobian of the affine approximation of the projective transformation [S1]. The color $C(\mathbf{p})$ at pixel $\mathbf{p}$ is computed via front-to-back alpha compositing [S2]:
    $$ C(\mathbf{p}) = \sum_{i=1}^K \mathbf{c}_i \alpha_i \mathcal{G}_i^{\text{2D}}(\mathbf{p}) \prod_{j=1}^{i-1} (1 - \alpha_j \mathcal{G}_j^{\text{2D}}(\mathbf{p})) $$
    where $\mathcal{G}_i^{\text{2D}}(\mathbf{p})$ is the 2D Gaussian value at pixel $\mathbf{p}$, $\mathbf{c}_i$ is the color, and $\alpha_i$ is the opacity [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def feed_forward_gaussian_splatting(images, cameras, patch_size=16, cnn_channels=64, transformer_dim=512, num_heads=8, num_layers=12):
    """
    Predicts 3D Gaussian primitives from N RGB images.
    
    Args:
        images: List of N numpy arrays, shape (H, W, 3), dtype float32, normalized [0, 1].
        cameras: List of N camera objects with 'extrinsics' (4x4) and 'intrinsics' (3x3).
        patch_size: Integer, size of non-overlapping patches.
        cnn_channels: Integer, output channels of the shallow CNN.
        transformer_dim: Integer, hidden dimension of the Transformer.
        num_heads: Integer, number of attention heads.
        num_layers: Integer, number of Transformer layers.
        
    Returns:
        gaussians: List of dictionaries, each containing 'mu', 'cov', 'alpha', 'sh_coeffs', 'features'.
    """
    N = len(images)
    H, W, _ = images[0].shape
    
    # Step 1: Extract per-pixel features (RGB + Plücker embeddings)
    # Plücker embeddings require ray origin and direction for each pixel.
    # Assuming a helper function `compute_plucker_embeddings` exists.
    pixel_features = []
    for i in range(N):
        img = images[i]
        # Compute ray origins and directions from camera extrinsics/intrinsics
        origins, directions = compute_ray_data(cameras[i], H, W)
        # Compute Plücker embeddings (6D vector per pixel)
        plucker = compute_plucker_embeddings(origins, directions)
        # Concatenate RGB (3D) and Plücker (6D) -> 9D vector
        # Expand plucker to match image shape (H, W, 6)
        plucker_expanded = np.repeat(plucker[:, :, np.newaxis, :], 3, axis=2).reshape(H, W, 6)
        # Concatenate along channel dimension
        combined = np.concatenate([img, plucker_expanded], axis=-1)
        pixel_features.append(combined)
        
    # Step 2: Group into patches and encode with shallow CNN
    # Reshape each view's features into patches
    # Shape: (N, H/p, W/p, p, p, 9)
    patches = []
    for i in range(N):
        feat = pixel_features[i]
        # Reshape to extract patches
        # Note: Implementation detail for patch extraction varies. 
        # Assuming a standard reshape and transpose to get (N, num_patches, p*p*9)
        num_patches_h = H // patch_size
        num_patches_w = W // patch_size
        num_patches = num_patches_h * num_patches_w
        
        # Reshape to (N, num_patches_h, num_patches_w, p, p, 9)
        reshaped = feat.reshape(N, num_patches_h, patch_size, num_patches_w, patch_size, 9)
        # Transpose to (N, num_patches_h, num_patches_w, p, p, 9) -> (N, num_patches, p*p*9)
        # This is a simplified representation; actual implementation uses unfold or similar
        patch_data = reshaped.reshape(N, num_patches, patch_size * patch_size * 9)
        patches.append(patch_data)
        
    # Stack patches from all views: (N, num_patches, patch_size*patch_size*9)
    all_patches = np.stack(patches, axis=0)
    
    # Apply shallow CNN to each patch
    # CNN input: (N * num_patches, patch_size*patch_size*9)
    # CNN output: (N * num_patches, cnn_channels)
    batch_size = N * num_patches
    flat_patches = all_patches.reshape(batch_size, -1)
    cnn_output = shallow_cnn(flat_patches, channels=cnn_channels)
    
    # Reshape back to (N, num_patches, cnn_channels)
    patch_features = cnn_output.reshape(N, num_patches, cnn_channels)
    
    # Step 3: Flatten into token sequence
    # Shape: (N * num_patches, cnn_channels)
    tokens = patch_features.reshape(-1, cnn_channels)
    
    # Step 4: Process with Multi-View Transformer
    # Transformer expects sequence of tokens. 
    # Note: Multi-view attention requires positional encodings that encode view index and spatial position.
    # Assuming `multi_view_transformer` handles this internally or requires explicit PE.
    latents = multi_view_transformer(tokens, dim=transformer_dim, num_heads=num_heads, num_layers=num_layers)
    
    # Step 5: Linear projection to Gaussian primitives
    # Linear layer maps transformer_dim to the dimensionality of Gaussian parameters + features
    # Gaussian params: mu (3) + cov (6, since symmetric PSD) + alpha (1) + sh_coeffs (variable) + features (variable)
    # For simplicity, assume a fixed output dimension D_out
    D_out = 3 + 6 + 1 + 27 + 32 # Example: mu(3) + cov(6) + alpha(1) + SH(order 3, 27 coeffs) + features(32)
    linear_layer = Linear(transformer_dim, D_out)
    predictions = linear_layer(latents)
    
    # Step 6: Parse predictions into Gaussian primitives
    gaussians = []
    for k in range(predictions.shape[0]):
        pred = predictions[k]
        mu = pred[0:3]
        cov_flat = pred[3:9]
        alpha = pred[9]
        sh_coeffs = pred[10:37]
        features = pred[37:]
        
        # Reconstruct covariance matrix from flat vector
        # Assuming cov_flat contains the 6 unique elements of the upper triangle
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
*   **Patch Size:** The patch size $p$ must divide $H$ and $W$ evenly. If not, padding or cropping is required [S2].
*   **Plücker Embeddings:** Accurate computation of Plücker embeddings requires precise camera intrinsics and extrinsics [S2].

## Operating characteristics
*   **Time Complexity:** The dominant cost is the multi-view Transformer, which is $O((N \cdot \frac{HW}{p^2})^2 \cdot d^2)$ for self-attention, where $d$ is the hidden dimension. This is significantly faster than iterative optimization, which scales with the number of optimization steps and scene complexity.
*   **Memory Complexity:** Memory usage is dominated by the Transformer's attention maps and the stored Gaussian primitives. The number of primitives is $N \cdot \frac{HW}{p^2}$, which can be large for high-resolution images.
*   **Performance:** Enables real-time novel view synthesis after a single forward pass. Quality is competitive with NeRF but may lack the fine detail of iteratively optimized 3DGS [S1].
*   **Strengths:** Fast reconstruction, no per-scene training required, handles sparse views.
*   **Limitations:** Quality depends on the generalization capability of the Transformer. May struggle with complex geometries or textures not seen during training.

## Background
Feed-forward 3D Gaussian Splatting builds on [Gaussian Splatting](946-gaussian-splatting.md) [S1]. The specific feed-forward reconstruction model described here is based on the architecture from GS-LRM [S2]. It was proposed to address the computational cost of iterative Gaussian Splatting optimization.

## Sources
[S1] DisC-GS: Discontinuity-aware Gaussian Splatting, https://arxiv.org/abs/2405.15196v2, fetched 2026-10-26
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
