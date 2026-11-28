# Parallel Differentiable Rasterization

Also called: F_HDR / F_LDR, HDR-GS rasterization

## Overview
Parallel Differentiable Rasterization is a rendering method that jointly produces High Dynamic Range (HDR) and Low Dynamic Range (LDR) images from a set of 3D Gaussians using two parallel rasterization pipelines, $F_{HDR}$ and $F_{LDR}$ [S1]. It solves the problem of simultaneous HDR/LDR view synthesis by decoupling the color representation (HDR vs. LDR) while sharing the geometric projection and alpha-blending logic [S1]. It differs from standard differentiable rasterization by explicitly maintaining two distinct color streams for the same geometric primitives to handle wide dynamic range scenes without tone-mapping artifacts during training [S1].

## How it works
**Inputs:**
*   Set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$, where each $G_i$ is defined by center $\mu_i \in \mathbb{R}^3$, 3D covariance $\Sigma_i \in \mathbb{R}^{3 \times 3}$, intrinsic opacity $\alpha_i \in \mathbb{R}$, and HDR color $c_{hi} \in \mathbb{R}^3$ [S1].
*   Camera extrinsic matrix $M_{ext} \in \mathbb{R}^{4 \times 4}$ and intrinsic matrix $M_{int} \in \mathbb{R}^{3 \times 4}$ [S1].
*   LDR exposure time $\Delta t$ [S1].
*   LDR color parameters $c_{li}(\Delta t)$, derived from $c_{hi}$ via an MLP (fully connected $\to$ ReLU $\to$ fully connected $\to$ sigmoid) [S1].

**Outputs:**
*   Rendered HDR image $I_h \in \mathbb{R}^{H \times W \times 3}$ [S1].
*   Rendered LDR image $I_l(\Delta t) \in \mathbb{R}^{H \times W \times 3}$ [S1].

**Data Structures:**
*   **Gaussian Attributes:** Stored in arrays of size $N$. Fields: $\mu_i$ (vec3), $\Sigma_i$ (mat3x3), $\alpha_i$ (float), $c_{hi}$ (vec3), $c_{li}$ (vec3).
*   **Tile Grid:** Image divided into non-overlapping tiles (e.g., $16 \times 16$ pixels). Each tile contains a list of indices of Gaussians whose 2D bounding boxes overlap the tile [S1].
*   **Sorted List:** Within each tile, Gaussian indices are sorted by view-space depth $v_{i,z}$ (front-to-back) [S1].

**Steps:**

1.  **Project Centers:** Transform the 3D center $\mu_i$ to camera coordinates $v_i$ and then to image coordinates $u_i$ [S1].
    $$ \tilde{e}_i = [v_i, 1]^\top = M_{ext} [\mu_i, 1]^\top $$
    $$ \tilde{u}_i = [u_i, 1]^\top = M_{int} v_i $$
    where $v_i \in \mathbb{R}^3$ and $u_i \in \mathbb{R}^2$ [S1].

2.  **Transform Covariance:** Transform the 3D covariance $\Sigma_i$ to the camera coordinate system $\Sigma'_i$ using the viewing transformation $W_i$ and the Jacobian $J_i$ of the affine approximation of the projection $M_{int}M_{ext}$ [S1].
    $$ \Sigma'_i = J_i W_i \Sigma_i W_i^\top J_i^\top $$
    where $W_i$ is the first $3 \times 3$ submatrix of $M_{ext}$ and $J_i \in \mathbb{R}^{3 \times 3}$ [S1].

3.  **Derive 2D Covariance:** Obtain the 2D covariance $\Sigma''_i \in \mathbb{R}^{2 \times 2}$ by removing the third row and column from $\Sigma'_i$ [S1].
    *   *Pitfall:* This approximation skips the depth component of the covariance, which may introduce artifacts in high-curvature regions or near depth discontinuities [S1].

4.  **Tiling and Sorting:**
    *   Compute the 2D bounding box of each Gaussian using $\Sigma''_i$ and $u_i$.
    *   Assign each Gaussian to all tiles whose pixels overlap with this bounding box [S1].
    *   For each tile, sort the assigned Gaussians by their view-space depth $v_{i,z}$ in ascending order (front-to-back) [S1].

5.  **Compute Opacity Contribution:** For each Gaussian $j$ overlapping a pixel $p$, calculate the spatial probability $P(x_j|\mu_j, \Sigma_j)$ at the intersection point $x_j$ (where the ray from the camera optical center through $p$ intersects the Gaussian) [S1].
    $$ P(x|\mu_i, \Sigma_i) = \exp\left(-\frac{1}{2}(x - \mu_i)^\top \Sigma_i^{-1} (x - \mu_i)\right) $$
    The effective opacity $\sigma_j$ is:
    $$ \sigma_j = \alpha_j P(x_j|\mu_j, \Sigma_j) $$
    where $x_j$ is the 3D point on the Gaussian surface intersected by the ray for pixel $p$ [S1].

6.  **Alpha Blending:** Render the pixel $p$ by blending the colors of all $N$ ordered Gaussians overlapping $p$ in the tile [S1].
    $$ I_h(p) = \sum_{j \in N} c_{hj} \sigma_j \prod_{k=1}^{j-1} (1 - \sigma_k) $$
    $$ I_l(p|\Delta t) = \sum_{j \in N} c_{lj}(\Delta t) \sigma_j \prod_{k=1}^{j-1} (1 - \sigma_k) $$
    where $c_{hj}$ is the HDR color and $c_{lj}(\Delta t)$ is the LDR color for the $j$-th Gaussian [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_differentiable_rasterization(Gaussians, M_ext, M_int, Delta_t):
    """
    Gaussians: List of N objects with attributes:
        mu: (3,) center
        Sigma: (3,3) covariance
        alpha: scalar opacity
        c_h: (3,) HDR color
    M_ext: (4,4) extrinsic matrix
    M_int: (3,4) intrinsic matrix
    Delta_t: float exposure time
    Returns: I_h (H,W,3), I_l (H,W,3)
    """
    N = len(Gaussians)
    H, W = image_height, image_width # Derived from M_int or config
    tile_size = 16 # Common tile size
    
    # Precompute LDR colors if not precomputed
    for g in Gaussians:
        g.c_l = apply_mlp_to_hdr_color(g.c_h, Delta_t) # MLP: FC->ReLU->FC->Sigmoid
        
    # 1. Project and Transform
    projected_gaussians = []
    for i, g in enumerate(Gaussians):
        # Homogeneous center
        mu_h = np.append(g.mu, 1.0)
        
        # World to Camera
        v_h = M_ext @ mu_h
        v = v_h[:3]
        
        # Camera to Image
        u_h = M_int @ v_h
        u = u_h[:2]
        
        # Covariance transformation
        W_i = M_ext[:3, :3]
        # J_i is Jacobian of affine approx of M_int @ M_ext. 
        # In practice, often approximated or computed via finite differences 
        # or analytical derivative of projection. 
        # Here we assume J_i is available or computed.
        J_i = compute_jacobian_affine(M_int, M_ext) 
        
        Sigma_prime = J_i @ (W_i @ g.Sigma @ W_i.T) @ J_i.T
        
        # 2D Covariance by skipping 3rd row/col
        Sigma_2d = Sigma_prime[:2, :2]
        
        projected_gaussians.append({
            'id': i,
            'u': u,
            'Sigma_2d': Sigma_2d,
            'v_z': v[2], # View space depth
            'alpha': g.alpha,
            'c_h': g.c_h,
            'c_l': g.c_l,
            'mu': g.mu,
            'Sigma': g.Sigma
        })
        
    # 2. Tiling and Sorting
    # Initialize tile lists
    tiles = {} # Key: (tile_x, tile_y), Value: list of projected_gaussians
    for pg in projected_gaussians:
        # Compute bounding box in pixel coordinates
        # Approximate bbox using 2 sigma of Sigma_2d
        bbox = compute_2d_bbox(pg['u'], pg['Sigma_2d'])
        
        # Determine tiles covered by bbox
        tile_indices = get_tile_indices(bbox, tile_size, W, H)
        
        for t_idx in tile_indices:
            if t_idx not in tiles:
                tiles[t_idx] = []
            tiles[t_idx].append(pg)
            
    # Sort each tile by depth (front-to-back)
    for t_idx in tiles:
        tiles[t_idx].sort(key=lambda g: g['v_z'])
        
    # 3. Rasterization
    I_h = np.zeros((H, W, 3))
    I_l = np.zeros((H, W, 3))
    
    for t_idx, tile_gaussians in tiles.items():
        tile_x, tile_y = t_idx
        for y in range(tile_y * tile_size, min((tile_y + 1) * tile_size, H)):
            for x in range(tile_x * tile_size, min((tile_x + 1) * tile_size, W)):
                # Ray from camera center through pixel (x,y)
                # This requires inverse projection. 
                # For 3DGS, the intersection x_j is often approximated 
                # or the Gaussian is treated as a volume.
                # The formula uses x_j, the intersection point.
                # In standard 2D splatting, we evaluate P at the pixel center 
                # or use the 2D Gaussian directly. 
                # The source specifies x_j is the intersection of the ray 
                # with the Gaussian point cloud.
                
                # Approximation: Use pixel center in camera space or 
                # assume x_j is close to the projected center for small Gaussians.
                # However, the formula explicitly uses P(x_j | mu, Sigma).
                # If we assume the Gaussian is evaluated at the pixel's ray intersection:
                
                # Compute ray direction for pixel (x,y)
                # This part depends on the specific camera model implementation.
                # For the purpose of the algorithm structure:
                ray = get_ray_for_pixel(x, y, M_int, M_ext)
                
                # Find intersection x_j with the Gaussian j
                # This is non-trivial for 3D Gaussians. 
                # Often, for differentiable rasterization in 3DGS, 
                # the 2D projection is used directly: sigma = alpha * exp(...)
                # where the exponent uses the 2D distance from u_j.
                # The source formula P(x|mu, Sigma) is 3D.
                # However, the projection step yields Sigma_2d.
                # Standard 3DGS uses the 2D Gaussian:
                # sigma = alpha * exp(-0.5 * (x-pixel - u)^T Sigma_2d^-1 (x-pixel - u))
                
                # Let's stick to the source's explicit formula structure:
                # I(p) = sum c_j * sigma_j * prod(1-sigma_k)
                # sigma_j = alpha_j * P(x_j | mu_j, Sigma_j)
                
                # Implementation note: 
                # If x_j is not explicitly computed, the 2D approximation is:
                # sigma_j = alpha_j * exp(-0.5 * dist^T Sigma_2d^-1 dist)
                
                cumulative_alpha = 0.0
                color_h = np.zeros(3)
                color_l = np.zeros(3)
                
                for pg in tile_gaussians:
                    # Calculate distance from pixel (x,y) to projected center u
                    dist_vec = np.array([x, y]) - pg['u']
                    
                    # Compute 2D Gaussian probability
                    # P_2d = exp(-0.5 * dist^T Sigma_2d^-1 dist)
                    # Note: Source says P(x|mu, Sigma) is 3D. 
                    # But Sigma_2d is derived from Sigma_3d.
                    # We use the 2D covariance for the 2D rasterization.
                    try:
                        inv_Sigma_2d = np.linalg.inv(pg['Sigma_2d'])
                        exponent = -0.5 * (dist_vec @ inv_Sigma_2d @ dist_vec)
                        prob = np.exp(exponent)
                    except:
                        prob = 0.0
                        
                    sigma_j = pg['alpha'] * prob
                    
                    # Blend
                    weight = sigma_j * (1.0 - cumulative_alpha)
                    color_h += pg['c_h'] * weight
                    color_l += pg['c_l'] * weight
                    cumulative_alpha += weight * (1.0 - (cumulative_alpha - weight)) # Simplified: cumulative_alpha += sigma_j * (1 - prev_cumulative)
                    
                    # Correct alpha accumulation:
                    # new_cumulative = 1 - (1 - prev) * (1 - sigma)
                    # weight = sigma * (1 - prev)
                    # This loop logic needs to match the product formula:
                    # prod_{k=1}^{j-1} (1 - sigma_k)
                    
                # Re-implement blending loop correctly
                cumulative_alpha = 0.0
                color_h = np.zeros(3)
                color_l = np.zeros(3)
                
                for pg in tile_gaussians:
                    dist_vec = np.array([x, y]) - pg['u']
                    try:
                        inv_Sigma_2d = np.linalg.inv(pg['Sigma_2d'])
                        exponent = -0.5 * (dist_vec @ inv_Sigma_2d @ dist_vec)
                        prob = np.exp(exponent)
                    except:
                        prob = 0.0
                        
                    sigma_j = pg['alpha'] * prob
                    
                    # Contribution: c_j * sigma_j * prod_{k<j} (1 - sigma_k)
                    # prod_{k<j} (1 - sigma_k) is (1 - cumulative_alpha)
                    contribution = sigma_j * (1.0 - cumulative_alpha)
                    
                    color_h += pg['c_h'] * contribution
                    color_l += pg['c_l'] * contribution
                    cumulative_alpha += contribution
                    
                I_h[y, x] = color_h
                I_l[y, x] = color_l
                
    return I_h, I_l

def apply_mlp_to_hdr_color(c_h, Delta_t):
    # Placeholder for the specific MLP architecture described in [S1]
    # FC -> ReLU -> FC -> Sigmoid
    # Input: c_h, Delta_t (possibly concatenated or used as bias)
    # Output: c_l
    pass

def compute_jacobian_affine(M_int, M_ext):
    # Jacobian of the affine approximation of M_int @ M_ext
    # This is a constant matrix for a given camera pose
    M_proj = M_int @ M_ext
    # Affine approximation usually involves the top 2 rows of the projection 
    # for the 2D mapping, but J_i in the source is 3x3 for the 3D->3D transform 
    # before dropping to 2D.
    # The source says J_i is Jacobian of affine approx of Mint Mext.
    # Since Mint Mext is 3x4, the affine part is the 3x3 rotation/scale.
    return M_proj[:3, :3] # Simplified assumption
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot T)$ where $N$ is the number of Gaussians and $T$ is the average number of Gaussians per tile/pixel. Sorting is $O(N \log N)$ globally or $O(N_t \log N_t)$ per tile.
*   **Memory Complexity:** $O(N)$ for Gaussian attributes plus $O(H \cdot W)$ for the output buffers and tile structures.
*   **Strengths:** Enables joint training of HDR and LDR representations, preserving high dynamic range information while providing standard LDR outputs. Parallel execution of $F_{HDR}$ and $F_{LDR}$ allows efficient use of GPU resources.
*   **Limitations:** The approximation of 2D covariance by skipping the third row/column may lead to inaccuracies in depth-dependent opacity calculations [S1]. The MLP for LDR color generation adds computational overhead during training [S1].
*   **When to use:** When the application requires both HDR and LDR outputs from the same scene representation, such as in HDR novel view synthesis or tone-mapping research.

## Background
Parallel Differentiable Rasterization was introduced in the context of HDR-GS (HDR Gaussian Splatting) [S1]. It addresses the limitation of standard 3D Gaussian Splatting, which typically operates in LDR space, by extending the rendering pipeline to handle HDR colors directly. The method builds on the shared principles of tiling, sorting, and alpha blending found in [Rendering Implementation and Acceleration](21069-rendering-implementation-and-acceleration.md).

## Sources
[S1] HDR-GS: Efficient High Dynamic Range Novel View Synthesis at 1000x Speed via Gaussian Splatting, https://arxiv.org/abs/2405.15125v4, fetched 2026-10-26
