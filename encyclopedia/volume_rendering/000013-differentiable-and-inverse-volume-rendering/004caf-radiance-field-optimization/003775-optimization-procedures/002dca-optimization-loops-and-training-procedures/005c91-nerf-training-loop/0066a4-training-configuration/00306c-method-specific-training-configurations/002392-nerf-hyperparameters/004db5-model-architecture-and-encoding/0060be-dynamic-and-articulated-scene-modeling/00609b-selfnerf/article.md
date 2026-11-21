# SelfNeRF

Also called: SelfNeRF: Fast Training NeRF for Human from Monocular Self-rotating Video

## Overview
SelfNeRF is a novel view synthesis method for human performance that trains from scratch using monocular self-rotating video inputs, achieving high-fidelity results in approximately twenty minutes [S1, unofficial]. It distinguishes itself from sibling methods like [MoCo-NeRF](16197-moco-nerf.md) and [A-NeRF](18108-a-nerf.md) by eliminating the need for multi-view inputs or explicit skeletal priors, instead relying on a surface-relative hash encoding to aggregate inter-frame information and accelerate training [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A monocular self-rotating video of a human performer consisting of $N_t$ frames.
*   **Output:** A trained neural radiance field model capable of rendering novel views of the performer.

**Data Structures:**
*   **Surface Shape $\{T_t\}$:** A set of point clouds representing the human surface for each frame $t \in \{0, \dots, N_t - 1\}$. These are recovered using external reconstruction methods (e.g., VideoAvatar [S1, unofficial] or SelfRecon [S1, unofficial]).
*   **Body Mask $\{M_t\}$:** Binary masks for each frame $t$, identifying the human region.
*   **Hash Grid:** A multi-resolution hash encoding structure storing high-dimensional feature vectors.
*   **MLP $F_\omega$:** A neural network with weights $\omega$ that maps encoded features to density $\sigma$ and color $c$.

**Numbered Steps:**
1.  **Preprocessing:** For the input video, recover the surface shape $T_t$ and body mask $M_t$ for each frame $t$ using existing reconstruction algorithms [S1, unofficial].
2.  **Surface-Relative Representation:** For a query point $\mathbf{x}$ at frame $t$:
    *   Find the $K$-nearest neighbor points on the current frame's point cloud $T_t$.
    *   Calculate the signed distances from $\mathbf{x}$ to these neighbors.
    *   Map these neighbors and their signed distances to the corresponding points and signed distances in the canonical space (Frame 0) [S1, unofficial].
3.  **Encoding:** Encode the resulting surface-relative representation using multi-resolution hash encoding to produce a high-dimensional feature vector [S1, unofficial].
4.  **Radiance Prediction:** Feed the encoded feature vector into the MLP $F_\omega$ to regress the density $\sigma$ and color $c$ [S1, unofficial].
5.  **Rendering:** Render the image using volume rendering to calculate the pixel color $C(\mathbf{r})$ for each ray $\mathbf{r}$ [S1, unofficial].
6.  **Optimization:** Train the model by minimizing photometric loss and geometric guidance loss [S1, unofficial].

**Formulas:**
The volume rendering equation for a ray $\mathbf{r}$ is [S1, unofficial]:
$$
C(\mathbf{r}) = \sum_{i=1}^{N} \alpha(\mathbf{x}_i) \left( \prod_{j<i} (1 - \alpha(\mathbf{x}_j)) \right) c(\mathbf{x}_i, \mathbf{v})
$$
where:
*   $\alpha(\mathbf{x}_i) = 1 - \exp(-\sigma(\mathbf{x}_i) \delta_i)$ is the transmittance-derived opacity.
*   $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{v}$ are the sample points on the ray.
*   $\delta_i = u_{i+1} - u_i$ is the distance between adjacent sample points.
*   $c(\mathbf{x}_i, \mathbf{v})$ is the color predicted by $F_\omega$.

**Parameters:**
*   $N_t$: Total number of frames in the input monocular video [S1, unofficial].
*   $K$: Number of nearest neighbors used to calculate the surface-relative representation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def selfnerf_render(ray_o, ray_d, frame_idx, surface_clouds, hash_grid, mlp, K):
    """
    Render a single ray for SelfNeRF.
    
    Args:
        ray_o: Origin of the ray.
        ray_d: Direction of the ray.
        frame_idx: Index of the current frame t.
        surface_clouds: List of point clouds {T_t} for all frames.
        hash_grid: Multi-resolution hash encoding structure.
        mlp: Neural network F_omega.
        K: Number of nearest neighbors.
        
    Returns:
        color: Rendered color for the ray.
    """
    # 1. Sample points along the ray
    # Note: Specific sampling strategy (e.g., uniform, stratified) is not detailed in sources.
    # Assume standard NeRF sampling is used [G].
    t_vals = sample_points(ray_o, ray_d) 
    x_samples = ray_o + t_vals[:, None] * ray_d
    
    # 2. Compute Surface-Relative Representation for each sample
    features = []
    for x in x_samples:
        # Get surface cloud for current frame
        current_cloud = surface_clouds[frame_idx]
        
        # Find K-nearest neighbors in current frame's cloud
        # Note: Exact NN search algorithm (e.g., KD-Tree) is implementation detail [G].
        k_neighbors, distances = find_k_nearest_neighbors(current_cloud, x, K)
        
        # Map neighbors and distances to canonical space (Frame 0)
        # Note: The source states "take the corresponding point of the k-nearest neighbor 
        # points on the canonical space". This implies a correspondence mapping exists 
        # between frame t and frame 0.
        canonical_neighbors, canonical_distances = map_to_canonical(k_neighbors, distances, frame_idx)
        
        # Encode the relative representation
        # Note: Hash encoding function details are abstracted [G].
        feature = hash_encode(canonical_neighbors, canonical_distances)
        features.append(feature)
        
    features = torch.stack(features)
    
    # 3. Predict density and color
    # Note: View direction v is constant for the ray.
    sigma, color = mlp(features, ray_d)
    
    # 4. Volume Rendering
    # Compute opacity alpha
    # Note: delta_i calculation requires t_vals[i+1] - t_vals[i]
    deltas = t_vals[1:] - t_vals[:-1]
    # Pad delta for last point if necessary, or assume N samples
    deltas = torch.cat([deltas, torch.zeros_like(deltas[:1])])
    
    alpha = 1.0 - torch.exp(-sigma * deltas)
    
    # Compute transmittance
    # T_i = product_{j<i} (1 - alpha_j)
    transmittance = torch.cumprod(1.0 - alpha, dim=0)
    transmittance = torch.cat([torch.ones_like(transmittance[:1]), transmittance[:-1]])
    
    # Composite color
    weights = alpha * transmittance
    color = torch.sum(weights[:, None] * color, dim=0)
    
    return color
```

## Operating characteristics
*   **Training Time:** Approximately 20 minutes for convergence [S1, unofficial].
*   **Input Requirements:** Monocular self-rotating video; no multi-view setup required [S1, unofficial].
*   **Strengths:** Fast training speed due to hash encoding; ability to train from scratch with minimal input [S1, unofficial].
*   **Limitations:** Relies on accurate surface reconstruction from monocular video, which can be challenging for complex motions or occlusions [S1, unofficial].
*   **When to use:** When only monocular video is available and rapid training is desired.
*   **When not to use:** When multi-view data is available and higher fidelity is required, or when explicit skeletal animation is needed.

## Background
SelfNeRF was proposed by Bo Peng, Jun Hu, Jingtao Zhou, and Juyong Zhang from the University of Science and Technology of China [S1, unofficial]. It was published on arXiv in October 2022 [S1, unofficial]. The method addresses the challenge of high-fidelity novel view synthesis for human performance using only monocular inputs, a common scenario in practical applications like sports broadcasts and VR/AR [S1, unofficial].

## Sources
[S1] SelfNeRF: Fast Training NeRF for Human from Monocular Self-rotating Video, https://doi.org/10.48550/arxiv.2210.01651, fetched 2026-10-30 (unofficial)
