# Parallel Differentiable Rasterization

Also called: F_HDR / F_LDR, HDR-GS rasterization

## Overview
Parallel Differentiable Rasterization is a rendering technique that jointly generates High Dynamic Range (HDR) and Low Dynamic Range (LDR) views from a single set of 3D Gaussian primitives by employing two parallel differentiable rasterization pipelines ($F_{HDR}$ and $F_{LDR}$) [S1]. It distinguishes itself from standard 3D Gaussian Splatting by decoupling the color representation into HDR base values and LDR exposure-dependent values, allowing simultaneous optimization and rendering of both radiance ranges without tone-mapping artifacts during the differentiable process [S1].

## How it works
**Inputs:**
*   Gaussian point cloud parameters: $\{\mu_i, \Sigma_i, \alpha_i, c_{hi}\}_{i=1}^N$, where $\mu_i \in \mathbb{R}^3$ is the center, $\Sigma_i \in \mathbb{R}^{3 \times 3}$ is the covariance, $\alpha_i \in [0,1]$ is the intrinsic opacity, and $c_{hi} \in \mathbb{R}^3$ is the HDR color.
*   Camera matrices: Extrinsic $M_{ext} \in \mathbb{R}^{4 \times 4}$ and Intrinsic $M_{int} \in \mathbb{R}^{3 \times 4}$.
*   LDR exposure time $\Delta t$.

**Outputs:**
*   Rendered HDR image $I_h \in \mathbb{R}^{H \times W \times 3}$.
*   Rendered LDR image $I_l(\Delta t) \in \mathbb{R}^{H \times W \times 3}$.

**Data Structures:**
*   **Tiles:** The image plane is divided into non-overlapping tiles.
*   **Gaussian List per Tile:** A list of Gaussian indices assigned to each tile, sorted by view-space depth.

**Steps:**

1.  **Probability Evaluation:** For a point $x \in \mathbb{R}^3$, compute the probability density of the $i$-th Gaussian:
    $$P(x|\mu_i, \Sigma_i) = \exp\left(-\frac{1}{2}(x - \mu_i)^\top \Sigma_i^{-1} (x - \mu_i)\right)$$ [S1]

2.  **Coordinate Transformation:**
    *   Transform the Gaussian center $\mu_i$ from world to camera coordinates $v_i$ and then to image coordinates $u_i$ using homogeneous coordinates:
        $$\tilde{e}_i = [v_i, 1]^\top = M_{ext} [\mu_i, 1]^\top$$
        $$\tilde{u}_i = [u_i, 1]^\top = M_{int} v_i$$ [S1]
    *   Transform the 3D covariance $\Sigma_i$ to the camera coordinate system:
        $$\Sigma'_i = J_i W_i \Sigma_i W_i^\top J_i^\top$$
        where $J_i \in \mathbb{R}^{3 \times 3}$ is the Jacobian of the affine approximation of the projective transformation $M_{int}M_{ext}$, and $W_i \in \mathbb{R}^{3 \times 3}$ is the viewing transformation (first three rows and columns of $M_{ext}$) [S1].

3.  **2D Covariance Derivation:**
    *   Derive the 2D covariance matrix $\Sigma''_i \in \mathbb{R}^{2 \times 2}$ by directly skipping the third row and column of $\Sigma'_i$ [S1].

4.  **Tiling and Sorting:**
    *   Divide the 2D image into non-overlapping tiles.
    *   Assign each 3D Gaussian to the tiles where its 2D projection (defined by $u_i$ and $\Sigma''_i$) overlaps.
    *   Within each tile, sort the assigned Gaussians by view-space depth [S1].

5.  **Differentiable Blending:**
    *   For each pixel $p$, identify the set of $N$ ordered Gaussians overlapping the pixel in the corresponding tile.
    *   Compute the opacity contribution $\sigma_j$ for the $j$-th Gaussian:
        $$\sigma_j = \alpha_j P(x_j|\mu_j, \Sigma_j)$$
        where $x_j$ is the intersection point between the ray from the camera optical center through pixel $p$ and the $j$-th Gaussian [S1].
    *   Compute the LDR color $c_{lj}(\Delta t)$ from the HDR color $c_{hj}$ using an MLP. The source specifies that channels of $c_{hi}$ undergo an independent MLP containing a fully connected layer, a ReLU activation, another fully connected layer, and a sigmoid activation to produce $c_{li}$ [S1].
    *   Render the HDR image $I_h(p)$ and LDR image $I_l(p|\Delta t)$ via alpha blending:
        $$I_h(p) = \sum_{j \in N} c_{hj} \sigma_j \prod_{k=1}^{j-1} (1 - \sigma_k)$$
        $$I_l(p|\Delta t) = \sum_{j \in N} c_{lj}(\Delta t) \sigma_j \prod_{k=1}^{j-1} (1 - \sigma_k)$$ [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_differentiable_rasterization(gaussians, M_ext, M_int, delta_t):
    """
    gaussians: List of Gaussian objects with attributes:
               mu (3,), Sigma (3,3), alpha (float), c_hi (3,)
    M_ext: (4, 4) extrinsic matrix
    M_int: (3, 4) intrinsic matrix
    delta_t: float, exposure time for LDR
    Returns: I_h (H, W, 3), I_l (H, W, 3)
    """
    H, W = image_height, image_width
    tile_size = 16 # Typical tile size, not specified in source [G]
    num_tiles_x = (W + tile_size - 1) // tile_size
    num_tiles_y = (H + tile_size - 1) // tile_size
    
    # Initialize tile lists
    # tile_gaussians[y][x] = list of gaussian indices
    tile_gaussians = [[[] for _ in range(num_tiles_x)] for _ in range(num_tiles_y)]
    
    # Precompute transformed Gaussians
    transformed_gaussians = []
    for i, g in enumerate(gaussians):
        # 1. Probability function definition (used later)
        # P(x|mu, Sigma) = exp(-0.5 * (x-mu)^T Sigma^-1 (x-mu))
        
        # 2. Coordinate Transformation
        # Homogeneous center
        mu_h = np.append(g.mu, 1.0)
        # World to Camera
        v_h = M_ext @ mu_h
        v = v_h[:3] / v_h[3] # Perspective division
        
        # Camera to Image
        u_h = M_int @ v_h
        u = u_h[:2] / u_h[2] # Perspective division
        
        # 3. Covariance Transformation
        # J_i: Jacobian of affine approximation of M_int @ M_ext
        # W_i: First 3 rows/cols of M_ext
        # Note: Source defines Sigma'_i = J_i W_i Sigma_i W_i^T J_i^T
        # Implementation detail: J_i is typically derived from the projection matrix.
        # For a standard pinhole camera, J_i relates to the derivative of projection.
        # We assume a helper function get_jacobian(M_int, M_ext, v) exists or is computed.
        J_i = get_jacobian(M_int, M_ext, v) 
        W_i = M_ext[:3, :3]
        
        Sigma_prime = J_i @ (W_i @ g.Sigma @ W_i.T) @ J_i.T
        
        # 4. 2D Covariance
        # Skip third row and column
        Sigma_double_prime = Sigma_prime[:2, :2]
        
        transformed_gaussians.append({
            'index': i,
            'u': u,
            'Sigma_2d': Sigma_double_prime,
            'v_depth': v[2], # View space depth for sorting
            'alpha': g.alpha,
            'c_hi': g.c_hi,
            'mu_3d': g.mu,
            'Sigma_3d': g.Sigma
        })
        
        # 5. Tiling and Assignment
        # Determine tile coordinates covering the Gaussian
        # Approximate bounding box using Sigma_2d eigenvalues or fixed radius
        # Source says: "assigned to the tiles where their 2D projections cover"
        # We iterate over tiles that overlap the Gaussian's bounding box
        tile_coords = get_tiles_covering_gaussian(u, Sigma_double_prime, tile_size, W, H)
        for ty, tx in tile_coords:
            tile_gaussians[ty][tx].append(i)

    # Initialize output images
    I_h = np.zeros((H, W, 3))
    I_l = np.zeros((H, W, 3))
    
    # 6. Blending
    for ty in range(num_tiles_y):
        for tx in range(num_tiles_x):
            tile_idx_list = tile_gaussians[ty][tx]
            if not tile_idx_list:
                continue
            
            # Sort by view space depth
            tile_idx_list.sort(key=lambda idx: transformed_gaussians[idx]['v_depth'])
            
            # Define pixel range for this tile
            y_start, y_end = ty * tile_size, min((ty + 1) * tile_size, H)
            x_start, x_end = tx * tile_size, min((tx + 1) * tile_size, W)
            
            # For each pixel in the tile
            for y in range(y_start, y_end):
                for x in range(x_start, x_end):
                    # Ray intersection points x_j
                    # Source: x_j is the j-th intersection 3D point between the ray... and the Gaussian
                    # In standard 3DGS, x_j is often approximated as the Gaussian center projected,
                    # or the intersection of the ray with the Gaussian ellipsoid.
                    # The formula uses P(x_j | mu, Sigma).
                    # We need to find x_j for each Gaussian in the tile that covers pixel (x,y).
                    
                    # Accumulators
                    alpha_acc = 1.0
                    color_h = np.zeros(3)
                    color_l = np.zeros(3)
                    
                    for idx in tile_idx_list:
                        g = transformed_gaussians[idx]
                        
                        # Check if Gaussian covers pixel (x,y)
                        # Using Mahalanobis distance with Sigma_2d
                        # Point in local Gaussian space? No, x_j is in world/camera space.
                        # Standard 3DGS rasterization checks if pixel is within bounding box or uses density.
                        # Here, we compute sigma_j = alpha_j * P(x_j | mu_j, Sigma_j)
                        
                        # Determine x_j: Intersection of ray from camera center through (x,y) with Gaussian j
                        # Ray origin: Camera center (inverse of M_ext)
                        # Ray direction: M_int^-1 * [x, y, 1] transformed to camera space
                        # This is complex. Simplified approach often used in 3DGS:
                        # x_j is approximated as the Gaussian center mu_j if the ray passes close,
                        # or computed via ellipsoid intersection.
                        # Given the source formula, we assume x_j is computed.
                        x_j = compute_ray_gaussian_intersection(g['mu_3d'], g['Sigma_3d'], camera_center, ray_direction)
                        
                        # Compute Probability
                        diff = x_j - g['mu_3d']
                        prob = np.exp(-0.5 * diff.T @ np.linalg.inv(g['Sigma_3d']) @ diff)
                        
                        sigma_j = g['alpha'] * prob
                        
                        if sigma_j <= 0:
                            continue
                            
                        # Compute LDR color from HDR color via MLP
                        # Source: channels of c_hi undergo independent MLP (fc, relu, fc, sigmoid)
                        c_lj = apply_color_mlp(g['c_hi'], delta_t)
                        
                        # Alpha blending
                        weight = sigma_j * alpha_acc
                        color_h += weight * g['c_hi']
                        color_l += weight * c_lj
                        alpha_acc *= (1.0 - sigma_j)
                        
                        if alpha_acc < 1e-4: # Early exit optimization
                            break
                    
                    I_h[y, x] = color_h
                    I_l[y, x] = color_l

    return I_h, I_l

def apply_color_mlp(c_hi, delta_t):
    """
    Applies the MLP described in source to convert HDR color to LDR color.
    """
    # Independent MLP per channel? Source says "channels ... undergo an independent MLP"
    # This implies 3 separate MLPs or one MLP with 3 outputs?
    # "independent MLP containing a fully connected (fc) layer, a ReLU activation, 
    # an fc layer, and a sigmoid activation"
    # Assuming weights are learned or fixed. Since not provided, we mark as [G] placeholder for structure.
    # In implementation, these weights would be parameters.
    # For the purpose of this article, we define the structure.
    
    # Channel 0
    h0 = np.dot(W1_0, c_hi) + b1_0 # fc layer
    h0 = np.maximum(0, h0) # ReLU
    c_l0 = sigmoid(np.dot(W2_0, h0) + b2_0) # fc + sigmoid
    
    # Channel 1
    h1 = np.dot(W1_1, c_hi) + b1_1
    h1 = np.maximum(0, h1)
    c_l1 = sigmoid(np.dot(W2_1, h1) + b2_1)
    
    # Channel 2
    h2 = np.dot(W1_2, c_hi) + b1_2
    h2 = np.maximum(0, h2)
    c_l2 = sigmoid(np.dot(W2_2, h2) + b2_2)
    
    return np.array([c_l0, c_l1, c_l2])

def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-x))
```

## Operating characteristics
*   **Time Complexity:** Dominated by the rasterization step, which is $O(N \cdot T)$ where $N$ is the number of Gaussians and $T$ is the number of tiles, assuming efficient tiling and sorting. The MLP application adds a constant factor per Gaussian per pixel.
*   **Memory Complexity:** Requires storing the Gaussian parameters and the tile lists. The parallel nature does not significantly increase memory overhead compared to single-pass rasterization, as the two passes share the same geometric data.
*   **Strengths:** Enables simultaneous HDR and LDR rendering, which is crucial for training on HDR datasets and generating LDR outputs for standard displays without post-processing tone mapping.
*   **Limitations:** Requires the additional MLP for LDR color conversion, adding slight computational overhead. The derivation of the 2D covariance by skipping the third row/column is an approximation that may introduce artifacts in extreme viewing angles [S1].

## Background
Parallel Differentiable Rasterization was introduced in the context of HDR-GS (HDR Gaussian Splatting) to address the lack of suitable 3DGS-based methods for 3D HDR imaging, particularly given that existing multiview HDR datasets often provide camera poses in the Normalized Device Coordinate (NDC) system, which is unsuitable for standard 3DGS [S1].

## Sources
[S1] HDR-GS: Efficient High Dynamic Range Novel View Synthesis at 1000x Speed via Gaussian Splatting, https://arxiv.org/abs/2405.15125v4, fetched 2026-10-26
