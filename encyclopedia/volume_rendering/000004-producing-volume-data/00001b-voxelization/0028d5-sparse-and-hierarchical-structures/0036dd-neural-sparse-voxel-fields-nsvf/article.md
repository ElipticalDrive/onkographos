# Neural Sparse Voxel Fields (NSVF)

Also called: Sparse Voxel Fields
## Overview
Neural Sparse Voxel Fields (NSVF) is a hybrid neural representation that models a scene as a set of voxel-bounded implicit functions organized in a sparse voxel octree [S2, S4, unofficial]. Unlike Neural Radiance Fields (NeRF), which uses a single monolithic implicit function for the entire scene, NSVF assigns a local implicit function to each non-empty voxel, enabling efficient sampling by skipping empty space [S2, S5, unofficial]. This approach offers approximately 10 times faster inference than NeRF while maintaining photorealistic quality [S2, S4, unofficial]. It solves the problem of inefficient ray sampling in large or sparse scenes by restricting computation to occupied regions [S6, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of posed 2D images (camera poses and RGB values), an initial sparse voxel grid (bounding box subdivided into ~1000 voxels or initialized from coarse geometry) [S4, unofficial].
*   **Outputs:** Rendered RGB images and depth maps for novel viewpoints [S4, unofficial].

**Data Structures**
*   **Sparse Voxel Octree:** Stores only non-empty voxels. Each voxel $V_i$ has 8 vertices.
*   **Vertex Features:** Each vertex stores a learnable feature vector $\tilde{g}_i \in \mathbb{R}^d$ (default $d=32$) [S1, S2, unofficial].
*   **Voxel Embedding:** Computed via trilinear interpolation of the 8 corner vertex features, followed by positional encoding (Fourier features) [S2, S4, unofficial].

**Rendering Algorithm**
Rendering is a two-step procedure: ray-voxel intersection and ray-marching inside voxels [S2, S4, unofficial].

1.  **Ray-Voxel Intersection:** For each ray, perform an Axis-Aligned Bounding Box (AABB) intersection test against the sparse voxel octree to identify intersected voxels [S2, S4, unofficial]. This efficiently skips empty space [S5, unofficial].
2.  **Ray-Marching:** For each intersected voxel, uniformly sample points along the ray segment inside the voxel [S4, unofficial].
3.  **Implicit Function Evaluation:** For each sample point $\mathbf{p}$ inside voxel $V_i$:
    *   Retrieve the 8 vertex features $\tilde{g}_i(\mathbf{p}^*_1), \dots, \tilde{g}_i(\mathbf{p}^*_8)$ of the containing voxel [S2, unofficial].
    *   Compute the voxel embedding $g_i(\mathbf{p})$ using trilinear interpolation $\chi$ and positional encoding $\zeta$:
        $$ g_i(\mathbf{p}) = \zeta \left( \chi \left( \tilde{g}_i (\mathbf{p}^*_1), \ldots, \tilde{g}_i (\mathbf{p}^*_8 ) \right) \right) $$
        where $\mathbf{p}^*_1, \ldots, \mathbf{p}^*_8$ are the 3D coordinates of the voxel vertices [S2, unofficial].
    *   Query the neural network $F_{\theta}^i$ with the embedding and ray direction $\mathbf{v}$ to predict density $\sigma$ and color $\mathbf{c}$:
        $$ F_{\theta}^i: (g_i (\mathbf{p}) , \mathbf{v}) \rightarrow (\mathbf{c}, \sigma) $$
4.  **Volume Rendering:** Accumulate color and transparency using the midpoint rule [S2, unofficial]. Early termination is applied when accumulated transparency $A$ drops below a threshold $\varepsilon = 0.01$ [S4, unofficial].

**Training and Optimization**
*   **Loss Function:**
    $$ \mathcal{L} = \sum_{(p_0, v) \in R} \| C(p_0, v) - C^*(p_0, v) \|^2_2 + \lambda \Omega (A(p_0, v)) $$
    where $R$ is a batch of rays, $C^*$ is ground-truth color, and $\Omega$ is a beta-distribution regularizer on accumulated transparency $A$ to keep it between 0 and 1 [S2, unofficial].
*   **Progressive Training:** Resources are allocated progressively. At specific iterations (e.g., 5K, 25K, 75K), the voxel size $l$ and ray-marching step size $\tau$ are halved [S4, unofficial]. Each voxel is subdivided into 8 sub-voxels, and new vertex features are initialized via trilinear interpolation of the original 8 vertices [S4, unofficial].
*   **Self-Pruning:** Empty voxels are pruned periodically (e.g., every 2.5K iterations). A voxel $V_i$ is pruned if:
    $$ \min_{j=1\ldots G} \exp (- \sigma (g_i (p_j))) > \gamma $$
    where $p_j$ are $G=16^3$ uniformly sampled points inside $V_i$, $\sigma$ is predicted density, and $\gamma = 0.5$ is the threshold [S2, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def render_nsvf(ray_origins, ray_directions, voxel_tree, vertex_features, model, params):
    """
    Render a batch of rays using Neural Sparse Voxel Fields.
    
    Args:
        ray_origins: Tensor (N_rays, 3)
        ray_directions: Tensor (N_rays, 3)
        voxel_tree: Sparse voxel octree structure supporting AABB intersection
        vertex_features: Dict mapping voxel_id -> Tensor (8, D) vertex embeddings
        model: Neural network taking (voxel_embedding, direction) -> (sigma, color)
        params: Dict containing N_samples, epsilon, etc.
        
    Returns:
        colors: Tensor (N_rays, 3)
        depths: Tensor (N_rays, 1)
    """
    N_rays = ray_origins.shape[0]
    N_samples = params.get('N_samples', 64)
    epsilon = params.get('epsilon', 0.01)
    
    # Initialize accumulation buffers
    accumulated_alpha = torch.zeros(N_rays, device=ray_origins.device)
    accumulated_color = torch.zeros(N_rays, 3, device=ray_origins.device)
    accumulated_depth = torch.zeros(N_rays, device=ray_origins.device)
    
    # 1. Ray-Voxel Intersection
    # Identify intersected voxels for each ray
    # Returns list of lists: intersected_voxels[r] = [voxel_id, start_t, end_t, ...]
    intersected_voxels = perform_ray_voxel_intersection(ray_origins, ray_directions, voxel_tree)
    
    for r in range(N_rays):
        voxels = intersected_voxels[r]
        if not voxels:
            continue
            
        # 2. Ray-Marching inside Voxels
        # Collect all sample points along the ray within intersected voxels
        sample_points = []
        sample_deltas = []
        
        current_t = 0.0
        for voxel_id, t_start, t_end in voxels:
            # Uniform sampling within the voxel segment
            # Number of samples proportional to voxel size or fixed per voxel
            # For simplicity, assume uniform distribution across the segment
            num_samples_voxel = N_samples // len(voxels) if len(voxels) > 0 else 1
            if num_samples_voxel < 1: num_samples_voxel = 1
            
            t_vals = torch.linspace(t_start, t_end, num_samples_voxel + 1, device=ray_origins.device)
            # Midpoints for sampling
            t_mid = (t_vals[:-1] + t_vals[1:]) / 2.0
            
            # Query points in world space
            points = ray_origins[r] + t_mid.unsqueeze(1) * ray_directions[r]
            sample_points.append(points)
            
            # Deltas (distances between samples)
            deltas = t_vals[1:] - t_vals[:-1]
            # Convert to world distance if direction is not unit length
            deltas = deltas * torch.norm(ray_directions[r])
            sample_deltas.append(deltas)
            
        if not sample_points:
            continue
            
        all_points = torch.cat(sample_points, dim=0)
        all_deltas = torch.cat(sample_deltas, dim=0)
        
        # 3. Evaluate Implicit Function
        # Determine which voxel each point belongs to
        # (In practice, this is done during intersection or via spatial lookup)
        # For each point, get voxel_id, interpolate features, embed, and query model
        
        # Placeholder for voxel lookup logic
        # voxel_ids_for_points = get_voxel_ids_for_points(all_points, voxel_tree)
        
        # Compute voxel embeddings
        # g_i(p) = zeta(chi(g_i(p1*), ..., g_i(p8*)))
        # Assuming helper function exists to retrieve and interpolate 8 corners
        voxel_embeddings = compute_voxel_embedding(all_points, voxel_id_for_points, vertex_features)
        
        # Query model: (embedding, direction) -> (sigma, color)
        # Direction is repeated for each sample
        dir_repeated = ray_directions[r].unsqueeze(0).expand(all_points.shape[0], -1)
        sigma, color = model(voxel_embeddings, dir_repeated)
        
        # Add noise to sigma for robustness
        noise = torch.randn_like(sigma) * params.get('noise_std', 0.0)
        sigma = torch.relu(sigma + noise)
        
        # 4. Volume Rendering Integration
        # Alpha: 1 - exp(-delta * sigma)
        alpha = 1.0 - torch.exp(-all_deltas * sigma)
        
        # Transmittance
        # T_k = product(1 - alpha_j) for j < k
        # We accumulate from front to back
        transmittance = 1.0 - accumulated_alpha[r]
        
        # Contribution
        contribution = alpha * transmittance
        accumulated_color[r] += contribution.unsqueeze(1) * color
        accumulated_depth[r] += contribution * (t_mid * torch.norm(ray_directions[r])) # Approx depth
        accumulated_alpha[r] += contribution
        
        # Early Termination
        if accumulated_alpha[r] > (1.0 - epsilon):
            break
            
    # Handle background if ray misses everything or transparency remains
    # (Simplified: assume white background if not fully opaque)
    # color = accumulated_color + (1 - accumulated_alpha) * background_color
    
    return accumulated_color, accumulated_alpha
```

**Parameters**
*   **Vertex Feature Dimension:** 32 [S1, unofficial].
*   **Fourier Features:** Maximum frequency $L=6$ [S4, unofficial].
*   **Pruning Threshold $\gamma$:** 0.5 [S2, unofficial].
*   **Early Termination $\varepsilon$:** 0.01 [S4, unofficial].
*   **Training Batch Size:** 32 images/batch, 2048 rays/image [S4, unofficial].
*   **Inference Chunk Size:** 1024*32 (32768) to manage memory [S3, unofficial].
*   **Progressive Halving Steps:** 5K, 25K, 75K iterations [S4, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero in Sampling:** When using importance sampling (if extended), add epsilon ($10^{-5}$) to weights before computing PDF [S3, unofficial].
*   **Embedding Size:** The 32-dim embedding with sinusoidal positional encoding leads to relatively large GPU time compared to lighter embeddings [S1, unofficial].
*   **Empty Space:** If no coarse geometry is provided, initialization must ensure the bounding box covers the scene with sufficient margin [S4, unofficial].

## Subtopics

### [SuRFNet](7895-surfnet.md)
SuRFNet is a 3D pipeline that distills multiple input image views into partial 3D Sparse Radiance Fields (SRFs) and learns within the 3D sparse voxel space to generate full SRFs from these partial inputs. It addresses the challenge of few-view novel view synthesis by ensuring multi-view consistency, particularly for out-of-distribution views, unlike 2D prior-based methods. It sets itself apart by operating directly on irregular volumetric sparse voxel structures using 3D sparse convolutions, rather than projecting 2D features or relying on dense volumetric grids.

## Operating characteristics
*   **Speed:** NSVF is approximately 10 times faster than NeRF at inference because it avoids sampling from empty space [S2, S4, unofficial].
*   **Memory:** Uses a sparse voxel octree, storing only non-empty voxels (10k~100k voxels are sufficient for photorealistic results) [S2, unofficial]. Parameter growth is additive rather than multiplicative, allowing scalability [S5, unofficial].
*   **Quality:** Achieves high PSNR and SSIM on synthetic and real datasets (e.g., Tanks & Temples, BlendedMVS) [S4, unofficial].
*   **Limitations:** Cannot handle complex backgrounds effectively in its basic form [S4, unofficial]. The 32-dim embedding with positional encoding increases GPU time [S1, unofficial].

## Background
NSVF was introduced by Liu et al. in 2020 [S3, S4, unofficial]. It extends the ideas of NeRF (Mildenhall et al. 2020) to a multi-resolution coarse-to-fine approach, single image to scene representation, and animated 3D objects by adding a time dimension [S3, unofficial]. It was presented at NeurIPS 2020 [S6, unofficial].

## Sources
[S1] Neural Sparse Voxel Fields, https://github.com/chaerinkong/paper-reviews/issues/85, fetched 2026-10-26 (unofficial)
[S2] Neural Sparse Voxel Fields | Takuma Yoneda, https://takuma.yoneda.xyz/posts/NSVF, fetched 2026-10-27 (unofficial)
[S3] GitHub - nitthilan/neural_sparse_voxel_field: Neural scene ..., https://github.com/nitthilan/neural_sparse_voxel_field, fetched 2026-10-28 (unofficial)
[S4] Learning Neural Sparse Voxel Fields for Free-viewpoint ..., https://irc.cs.sdu.edu.cn/2020-summer-school/static/LingjieLiu.pdf, fetched 2026-10-29 (unofficial)
[S5] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S6] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
