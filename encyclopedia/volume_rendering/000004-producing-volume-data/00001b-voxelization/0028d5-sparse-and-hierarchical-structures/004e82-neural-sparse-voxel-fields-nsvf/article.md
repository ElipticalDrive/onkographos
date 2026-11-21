# Neural Sparse Voxel Fields (NSVF)
Also called: NSVF (Neural Sparse Voxel Fields), Sparse Voxel Fields
## Overview
Neural Sparse Voxel Fields (NSVF) is a hybrid neural representation that models a scene as a set of voxel-bounded implicit fields organized in a sparse voxel octree [S3, unofficial][S5, unofficial]. Unlike Neural Radiance Fields (NeRF), which uses a single monolithic implicit function for the entire scene, NSVF partitions space into voxels, each containing a local implicit function mapping a point and ray direction to color and density [S3, unofficial][S5, unofficial]. This structure allows NSVF to skip sampling in empty space, making it approximately 10 times faster than NeRF at inference while supporting high-quality free-viewpoint rendering [S3, unofficial][S5, unofficial]. It builds on [Sparse and Hierarchical Structures](4391-sparse-and-hierarchical-structures.md) by using a sparse voxel grid for spatial embedding [S4, unofficial].

## How it works
**Inputs:** A set of posed 2D images (camera poses and images) [S5, unofficial].
**Outputs:** A rendered image for a new camera pose, consisting of accumulated color and depth [S5, unofficial].

**Data Structures:**
*   **Sparse Voxel Octree:** A hierarchical structure where nodes represent voxels [S3, unofficial]. Non-empty voxels store learnable feature vectors at their 8 vertices [S1].
*   **Vertex Features:** Each vertex of a voxel stores a learnable embedding vector $\tilde{g}_i \in \mathbb{R}^d$. The default dimension $d$ is 32 [S2, unofficial].
*   **Implicit Function:** For a voxel $V_i$, the function $F_{\theta}^i$ maps a point $\mathbf{p}$ and ray direction $\mathbf{v}$ to color $\mathbf{c}$ and density $\sigma$ [S3, unofficial].

**Parameters:**
*   $N_{samples}$: Number of coarse samples per ray. Default: 64 [S4, unofficial].
*   $N_{importance}$: Number of fine samples per ray. Default: 0 (no fine model) [S4, unofficial].
*   $\varepsilon$: Early termination threshold for accumulated transparency. Default: 0.01 [S5, unofficial].
*   $\gamma$: Pruning threshold. Default: 0.5 [S3, unofficial].
*   $L$: Maximum frequency for Fourier features. Default: 6 [S5, unofficial].
*   $\tau$: Ray-marching step size. Halved during progressive training [S3, unofficial].
*   $l$: Voxel size. Halved during progressive training [S3, unofficial].

**Algorithm Steps:**

1.  **Voxel Initialization:**
    *   Define an initial bounding box enclosing the scene with sufficient margin [S5, unofficial].
    *   Subdivide the bounding box into approximately 1000 voxels [S5, unofficial].
    *   If coarse geometry is available, initialize voxels by voxelizing the geometry; otherwise, initialize features randomly or via a specific method [S2, unofficial][S5, unofficial].

2.  **Feature Interpolation (Voxel Embedding):**
    *   For a query point $\mathbf{p}$ inside voxel $V_i$, identify the 8 vertices $\mathbf{p}^*_1, \ldots, \mathbf{p}^*_8 \in \mathbb{R}^3$ of $V_i$ [S3, unofficial].
    *   Retrieve the feature vectors $\tilde{g}_i(\mathbf{p}^*_1), \ldots, \tilde{g}_i(\mathbf{p}^*_8)$ stored at these vertices [S3, unofficial].
    *   Compute the voxel embedding $g_i(\mathbf{p})$ using trilinear interpolation $\chi$ and positional encoding $\zeta$ (e.g., Fourier features):
        $$ g_i(\mathbf{p}) = \zeta \left( \chi \left( \tilde{g}_i (\mathbf{p}^*_1), \ldots, \tilde{g}_i (\mathbf{p}^*_8 ) \right) \right) $$
        [S3, unofficial][S5, unofficial]

3.  **Ray-Voxel Intersection:**
    *   For each ray, perform an Axis-Aligned Bounding Box (AABB) intersection test to determine which voxels the ray intersects [S2, unofficial][S5, unofficial].
    *   This step efficiently skips empty space, as rays only proceed into intersected (non-empty) voxels [S5, unofficial].

4.  **Ray Marching Inside Voxels:**
    *   For each intersected voxel, sample points uniformly along the ray segment inside the voxel [S5, unofficial].
    *   Alternatively, use rejection sampling based on sparse voxels to create query points [S3, unofficial].
    *   For each sampled point $\mathbf{p}$, compute the voxel embedding $g_i(\mathbf{p})$ as defined in Step 2 [S5, unofficial].
    *   Concatenate $g_i(\mathbf{p})$ with the embedded ray direction $\mathbf{v}$ (if view-dependent color is used) [S4, unofficial].
    *   Pass the concatenated embedding through a shared Multi-Layer Perceptron (MLP) to predict volumetric density $\sigma$ and color $\mathbf{c}$ [S1][S3, unofficial].

5.  **Volume Rendering Integration:**
    *   Compute the distance $\delta$ between consecutive samples [S4, unofficial]. Convert to real-world distance by multiplying by the norm of the ray direction [S4, unofficial].
    *   Add noise to density predictions: $\sigma' = \sigma + \text{noise}$, where noise $\sim \mathcal{N}(0, \text{noise\_std}^2)$ [S4, unofficial].
    *   Compute alpha (opacity) for each sample:
        $$ \alpha = 1 - \exp(-\delta \cdot \max(0, \sigma')) $$
        [S4, unofficial]
    *   Accumulate color and depth using standard volume rendering equations with early termination:
        $$ C = \sum_{i} T_i \alpha_i \mathbf{c}_i, \quad T_i = \prod_{j < i} (1 - \alpha_j) $$
        [S4, unofficial]
    *   Terminate accumulation when the accumulated transparency $A = \sum T_i \alpha_i$ drops below $\varepsilon = 0.01$ [S2, unofficial][S5, unofficial].

6.  **Self-Pruning:**
    *   Periodically (e.g., every 2.5K iterations) prune empty voxels [S5, unofficial].
    *   A voxel $V_i$ is pruned if:
        $$ \min_{j=1\ldots G} \exp (- \sigma (g_i (p_j))) > \gamma $$
        where $p_j$ are $G=16^3$ uniformly sampled points inside $V_i$, and $\gamma = 0.5$ [S3, unofficial].

7.  **Progressive Training:**
    *   Train from coarse to fine resolution [S5, unofficial].
    *   At specific steps (e.g., 5K, 25K, 75K iterations), halve the voxel size $l$ and ray-marching step size $\tau$ [S5, unofficial].
    *   Subdivide each voxel into $2^3=8$ sub-voxels [S2, unofficial].
    *   Initialize the feature representations of the new vertices via trilinear interpolation of the original 8 vertices' features [S3, unofficial][S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def ray_aabb_intersect(origin, direction, aabb_min, aabb_max):
    """Standard slab method for ray-AABB intersection."""
    inv_dir = 1.0 / direction
    t1 = (aabb_min - origin) * inv_dir
    t2 = (aabb_max - origin) * inv_dir
    t_min = torch.where(t1 < t2, t1, t2)
    t_max = torch.where(t1 > t2, t1, t2)
    t_enter = torch.clamp(t_min.min(), min=0.0)
    t_exit = torch.clamp(t_max.max(), max=1e10)
    if t_enter > t_exit:
        return None, None
    return t_enter, t_exit

def get_local_coords(world_point, voxel_min, voxel_size):
    """Convert world coordinates to local [0,1]^3 coordinates."""
    return (world_point - voxel_min) / voxel_size

def trilinear_interpolate(features, p_local):
    """Interpolate features at 8 vertices given local coordinates p_local in [0,1]^3."""
    x, y, z = p_local.unbind()
    w000 = (1-x)*(1-y)*(1-z)
    w100 = x*(1-y)*(1-z)
    w010 = (1-x)*y*(1-z)
    w110 = x*y*(1-z)
    w001 = (1-x)*(1-y)*z
    w101 = x*(1-y)*z
    w011 = (1-x)*y*z
    w111 = x*y*z
    weights = torch.stack([w000, w100, w010, w110, w001, w101, w011, w111])
    return torch.sum(weights.unsqueeze(1) * features, dim=0)

def fourier_encoding(x, L=6):
    """Apply Fourier positional encoding up to frequency L."""
    out = [x]
    freq_bands = 2 ** torch.arange(L, device=x.device)
    for freq in freq_bands:
        out.append(torch.sin(math.pi * freq * x))
        out.append(torch.cos(math.pi * freq * x))
    return torch.cat(out, dim=-1)

def query_nsvf(voxel_features, p_local, ray_dir, L=6):
    """Query the NSVF implicit function for a point inside a voxel."""
    g_interp = trilinear_interpolate(voxel_features, p_local)
    g_encoded = fourier_encoding(g_interp, L)
    # Simplified MLP: Linear -> ReLU -> Linear -> ReLU (sigma) / Sigmoid (color)
    # In practice, this is a small shared MLP. We simulate it here.
    h = torch.relu(g_interp @ torch.randn(g_interp.shape[0], 32))
    out = h @ torch.randn(32, 4)
    sigma = torch.relu(out[..., 0])
    color = torch.sigmoid(out[..., 1:4])
    return sigma, color

def render_ray(ray_origin, ray_direction, voxels, L=6, epsilon=0.01):
    """Render a single ray using NSVF."""
    total_color = torch.zeros(3, device=ray_origin.device)
    total_transmittance = 1.0
    
    for voxel in voxels:
        t_min, t_max = ray_aabb_intersect(ray_origin, ray_direction, voxel.aabb_min, voxel.aabb_max)
        if t_min is None:
            continue
            
        num_samples = 64
        t_vals = torch.linspace(t_min, t_max, num_samples, device=ray_origin.device)
        # Compute deltas for uniform sampling
        deltas = torch.diff(t_vals)
        deltas = torch.cat([deltas, torch.tensor([t_max - t_vals[-1]], device=ray_origin.device)])
        
        # World positions for samples
        p_world = ray_origin.unsqueeze(0) + t_vals.unsqueeze(1) * ray_direction.unsqueeze(0)
        
        for i in range(num_samples):
            p_local = get_local_coords(p_world[i], voxel.min_corner, voxel.size)
            sigma, color = query_nsvf(voxel.features, p_local, ray_direction, L)
            
            delta_real = deltas[i] * torch.norm(ray_direction)
            alpha = 1.0 - torch.exp(-delta_real * sigma)
            
            weight = total_transmittance * alpha
            total_color += weight * color
            total_transmittance *= (1.0 - alpha)
            
            if total_transmittance < epsilon:
                break
                
    return total_color
```

## Subtopics

### [SuRFNet](7895-surfnet.md)
SuRFNet is a 3D pipeline that distills multiple input image views into partial 3D Sparse Radiance Fields (SRFs) and learns within the 3D sparse voxel space to generate full SRFs from these partial inputs. It addresses few-view novel view synthesis by ensuring multi-view consistency, particularly for out-of-distribution views, unlike 2D prior-based methods. It operates directly on irregular volumetric sparse voxel structures using 3D sparse convolutions, rather than projecting 2D features or relying on dense volumetric grids.

## Operating characteristics
**Time Complexity:** Inference is approximately 10 times faster than NeRF because it avoids sampling from empty space [S3, unofficial]. The ray-voxel intersection is efficient, especially for hierarchical octree structures [S3, unofficial]. However, the 32-dim vertex feature embedding with sinusoidal positional encoding leads to relatively large GPU time compared to lighter embeddings [S2, unofficial].

**Memory Complexity:** The method is more memory-intensive than NeRF due to storing feature vectors on a potentially dense voxel grid, although sparsity helps [S1]. The parameter growth is additive rather than multiplicative, allowing scalability to large spaces [S6, unofficial].

**Performance:**
*   **Synthetic Scenes:** Trained with 4 stages; real scenes with 3 stages [S3, unofficial].
*   **Voxel Count:** 10k~100k sparse voxels are enough for photorealistic results [S3, unofficial].
*   **Hardware:** Evaluated on Nvidia V100 GPUs [S5, unofficial].
*   **Scaling:** Scales well for large scenes due to spatial decomposition [S6, unofficial].

**Strengths:**
*   Fast inference due to empty space skipping [S3, unofficial][S5, unofficial].
*   Supports progressive training for coarse-to-fine detail [S5, unofficial].
*   Explicit geometry allows for spatial editing [S6, unofficial].

**Limitations:**
*   Cannot handle complex backgrounds well [S5, unofficial].
*   Memory intensive due to vertex feature storage [S1].
*   Pruning threshold $\gamma$ defaults to 0.5, and progressive halving occurs at 5K, 25K, and 75K iterations [S3, unofficial].

## Background
NSVF was introduced by Liu et al. in July 2020 [S1]. It extends the ideas of NeRF by incorporating a sparse voxel grid for spatial embedding [S4, unofficial]. The method was developed to improve training and rendering speed, making it more suited for distributed computing [S1]. It is used for free-viewpoint rendering, scene editing, and composition [S5, unofficial].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Neural Sparse Voxel Fields, https://github.com/chaerinkong/paper-reviews/issues/85, fetched 2026-10-26 (unofficial)
[S3] Neural Sparse Voxel Fields | Takuma Yoneda, https://takuma.yoneda.xyz/posts/NSVF, fetched 2026-10-27 (unofficial)
[S4] GitHub - nitthilan/neural_sparse_voxel_field: Neural scene ..., https://github.com/nitthilan/neural_sparse_voxel_field, fetched 2026-10-28 (unofficial)
[S5] Learning Neural Sparse Voxel Fields for Free-viewpoint ..., https://irc.cs.sdu.edu.cn/2020-summer-school/static/LingjieLiu.pdf, fetched 2026-10-29 (unofficial)
[S6] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S7] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
