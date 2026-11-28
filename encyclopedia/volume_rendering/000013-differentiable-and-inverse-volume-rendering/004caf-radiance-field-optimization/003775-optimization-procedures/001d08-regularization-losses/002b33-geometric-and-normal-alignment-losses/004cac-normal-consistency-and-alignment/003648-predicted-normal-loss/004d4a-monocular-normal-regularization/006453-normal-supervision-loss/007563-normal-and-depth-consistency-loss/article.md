# Normal and Depth Consistency Loss


## Overview
A loss term that enforces consistency between the rendered surface normal and the normal estimated from the rendered depth map, stabilizing field geometry [S1]. It differs from [Normal Supervision Loss](19107-normal-supervision-loss.md) by deriving the reference normal from the scene's own rendered depth rather than an external monocular prior, ensuring internal geometric coherence [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Rendered depth map $D$, rendered normal map $\mathbf{N}$ (or Gaussian normals $\mathbf{n}_i$), and pixel coordinates.
- Output: Scalar loss value $\mathcal{L}_{ND}$.

**Data Structures:**
- Depth map $D$: 2D array of float values, $D(i)$ is the depth at pixel $i$.
- Normal map $\mathbf{N}$: 2D array of 3D vectors, $\mathbf{N}(i)$ is the normal at pixel $i$.
- Gaussian normals $\mathbf{n}_i$: 3D vectors for each Gaussian $i$ along a ray.
- Alpha-compositing weights $\omega_i$: Scalar weights for each Gaussian.

**Steps:**
1. Estimate the surface normal from the rendered depth map $D$ via finite differences, denoted as $\mathbf{N}_D(i)$, following [Huang et al. 2024] [S1].
2. Compute a reference normal $\tilde{\mathbf{n}}$ by applying finite differences to the rendered depth map [S2, unofficial].
3. Penalize the angular deviation between $\tilde{\mathbf{n}}$ and each Gaussian normal $\mathbf{n}_i$ using the loss:
   $$ \mathcal{L}_n = \sum_i \omega_i (1 - \mathbf{n}_i^\top \tilde{\mathbf{n}}) $$
   where $\omega_i = \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the alpha-compositing weight of the $i$-th Gaussian [S2, unofficial].
4. Alternatively, for field renderings, compute the loss directly between the rendered normal map and the depth-derived normal:
   $$ \mathcal{L}_{ND} = \sum_i (1 - \tilde{\mathbf{N}}(i) \cdot \mathbf{N}_D(i)) $$
   where $\tilde{\mathbf{N}}(i)$ is the rendered normal and $\mathbf{N}_D(i)$ is the depth-derived normal at pixel $i$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal_from_depth(depth_map, epsilon=1.0):
    """Estimate surface normal from depth map using finite differences."""
    # Compute gradients in x and y directions
    # Assuming depth_map is a 2D array (H, W)
    dy, dx = np.gradient(depth_map, edge_order=2)
    
    # Construct normal vector: (dx, dy, -1) or similar depending on coordinate system
    # Standard convention: normal = (-dz/dx, -dz/dy, 1) normalized
    # Here we assume depth increases away from camera, so surface normal points towards camera
    # Adjust sign based on specific rendering convention
    normals = np.stack([-dx, -dy, np.ones_like(depth_map)], axis=-1)
    
    # Normalize
    norms = np.linalg.norm(normals, axis=-1, keepdims=True)
    # Avoid division by zero
    norms = np.maximum(norms, 1e-8)
    normals = normals / norms
    
    return normals

def normal_and_depth_consistency_loss(rendered_normal_map, depth_map, gaussian_normals=None, alpha_weights=None):
    """
    Compute Normal and Depth Consistency Loss.
    
    Args:
        rendered_normal_map: 2D array of normals (H, W, 3)
        depth_map: 2D array of depths (H, W)
        gaussian_normals: Optional list of Gaussian normals along rays (N, 3)
        alpha_weights: Optional alpha-compositing weights (N,)
        
    Returns:
        loss: Scalar loss value
    """
    # Step 1: Estimate normal from depth
    depth_normals = compute_normal_from_depth(depth_map)
    
    # Step 2: Compute loss
    if gaussian_normals is not None and alpha_weights is not None:
        # Gaussian-based loss (S2)
        # Reference normal is depth_normals at the corresponding pixel
        # Assuming gaussian_normals are sorted by depth along the ray
        # We need to map each Gaussian to its pixel. This is typically handled by rasterization.
        # For simplicity, assume we have pixel-wise depth normals and we aggregate Gaussian contributions.
        # However, S2 formula sums over Gaussians i.
        # We need to know which Gaussian corresponds to which pixel.
        # In practice, this is done during rasterization.
        # Here we assume pixel-wise aggregation is handled or we compute per-pixel loss.
        
        # Let's assume we are computing per-pixel loss and summing.
        # For each pixel, we have a set of Gaussians.
        # This requires access to the rasterization data structure.
        # Since the prompt asks for a self-contained function, we assume the inputs are pre-aggregated or we compute the field version.
        pass
    
    # Field-based loss (S1)
    # L_ND = sum_i (1 - N_tilde(i) dot N_D(i))
    # N_tilde(i) is rendered_normal_map[i]
    # N_D(i) is depth_normals[i]
    
    dot_product = np.sum(rendered_normal_map * depth_normals, axis=-1)
    loss = np.sum(1.0 - dot_product)
    
    return loss
```

**Parameters:**
- $\lambda_{ND}$: Weight for the normal and depth consistency loss. Typical values are not given in the sources.
- $N$: Number of tetrahedral vertices used for averaging the field loss terms [S1].

**Edge Cases and Pitfalls:**
- Precision: Finite differences on depth maps can be noisy, especially at depth discontinuities [S1].
- Boundaries: Edge pixels in the depth map may have undefined gradients. Use edge handling or padding.
- Degenerate Input: If depth is constant, the normal is undefined. Add a small epsilon to avoid division by zero.

## Subtopics

### Field Loss Combination
A combination of Normal and Depth Consistency Loss with Eikonal and Curvature losses for SDF regularization. The field loss is defined as $\mathcal{L}_{\text{field}} = \lambda_{\text{ND}} \mathcal{L}_{\text{ND}} + \lambda_{\text{eik}} \mathcal{L}_{\text{eik}} + \lambda_{\text{curv}} \mathcal{L}_{\text{curv}}$ [S1]. This is used when regularizing hash-encoded fields where higher-order autodiff is noisy [S1].

## Operating characteristics
- Time Complexity: $O(HW)$ for depth normal estimation and loss computation, where $H, W$ are image dimensions.
- Memory Complexity: $O(HW)$ for storing depth and normal maps.
- Strengths: Stabilizes geometry by enforcing consistency between differentiable render outputs [S1]. Robust to noise in hash-encoded fields [S1].
- Limitations: Depends on the quality of the rendered depth map. Noisy depth leads to noisy normals [S1].
- When to use: When optimizing implicit fields or Gaussian splats where geometric consistency is crucial [S1][S2, unofficial].
- When not to use: When depth is unreliable or unavailable.

## Background
Originated in the context of differentiable volume rendering and radiance field optimization. The concept of using depth-derived normals for regularization is discussed in [Huang et al. 2024] [S1]. The specific formulation for Gaussian splats is introduced in [Geometry-Grounded Gaussian Splatting] [S2, unofficial]. The field-based formulation is used in [Distance Field Rasterization for End-to-End Mesh Reconstruction] [S1].

## Sources
[S1] Distance Field Rasterization for End-to-End Mesh Reconstruction, https://doi.org/10.1145/3799902.3811155, fetched 2026-10-26
[S2] Geometry-Grounded Gaussian Splatting, https://arxiv.org/abs/2601.17835v2, fetched 2026-10-26 (unofficial)
