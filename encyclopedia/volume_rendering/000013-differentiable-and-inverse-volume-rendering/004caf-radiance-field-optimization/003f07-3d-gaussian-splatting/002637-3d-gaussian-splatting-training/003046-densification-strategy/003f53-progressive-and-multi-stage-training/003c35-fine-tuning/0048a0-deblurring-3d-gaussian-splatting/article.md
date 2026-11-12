# Deblurring 3D Gaussian Splatting

Also called: Deblur-3D-GS

## Overview
Deblurring 3D Gaussian Splatting is a real-time deblurring framework that reconstructs sharp 3D scenes from blurry input images by manipulating the covariance matrices and means of 3D Gaussians using a small Multi-Layer Perceptron (MLP) during training [S1]. Unlike volumetric rendering-based methods like Deblur-NeRF which require seconds per image, this method uses differentiable rasterization to achieve real-time inference speeds (>800 FPS) while handling both defocus and camera motion blur [S1, S3]. It addresses the sparse point clouds caused by blurry inputs through K-nearest-neighbor interpolation and depth-based pruning [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of blurry images, camera poses, and an initial sparse point cloud obtained from Structure-from-Motion (SfM) [S1].
*   **Outputs:** A set of optimized 3D Gaussians (position $x$, rotation $r$, scale $s$, opacity $\sigma$, spherical harmonics coefficients) that represent the sharp scene [S1].

**Data Structures:**
The method relies on the standard 3D Gaussian Splatting data structures [S1]:
*   **3D Gaussian $G_j$:** Characterized by position $x_j \in \mathbb{R}^3$, opacity $\sigma_j \in \mathbb{R}$, rotation $r_j$ (quaternion), scale $s_j \in \mathbb{R}^3$, and spherical harmonics coefficients [S1].
*   **MLP $F_\theta$:** A neural network with 4 layers. The first 3 layers are shared (64 hidden units, ReLU activation, Xavier initialization). The last layer consists of 3 single-layer heads producing deltas $\delta x, \delta r, \delta s$ [S1].
*   **Point Cloud $P$:** A set of 3D points with associated colors, initially from SfM [S1].

**Positional Encoding:**
Inputs to the MLP are processed using positional encoding $\gamma(p)$ [S1]:
$$ \gamma(p) = \big (\sin (2^k\pi p), \cos (2^k \pi p)\big )_{k=0}^{L-1} $$
where $L$ is the number of frequencies [S1].

**Algorithm Steps:**

1.  **Compensation for Sparse Point Cloud:**
    After $N_{st}$ training iterations, if the point cloud is sparse (common with blurry inputs or large depth of field), add extra points [S1]:
    *   Sample $N_p$ points from a uniform distribution $U(\alpha, \beta)$, where $\alpha$ and $\beta$ are the min/max position values of the existing point cloud $P$ [S1].
    *   For each new point $p$:
        *   Find $K$ nearest neighbors $P_{knn}$ from $P$ [S1].
        *   Discard neighbors if the distance to $p$ exceeds threshold $t_d$ [S1].
        *   If valid neighbors remain, interpolate color $p_c$ from them [S1].
        *   Add $p$ with color $p_c$ to $P$ [S1].
    *   *Note:* Prune Gaussians based on position to keep more Gaussians on the far plane, using a pruning threshold multiplier $w_p$ [S1].

2.  **Blur Modeling via MLP (Training Only):**
    The MLP $F_\theta$ transforms Gaussian attributes to simulate blur. The specific transformation depends on the blur type [S1].

    *   **Defocus Blur:**
        The MLP outputs scaling deltas $(\delta r_j, \delta s_j)$ [S1]:
        $$ (\delta r_j, \delta s_j) = F_\theta \Big (\gamma (x_j), r_j, s_j, \gamma (v) \Big ) $$
        where $v$ is the viewing direction [S1].
        Transform the attributes to obtain $\hat{r}_j$ and $\hat{s}_j$ [S1]:
        $$ \hat {r}_j=r_j\cdot \text {min}(1.0, ~\lambda _s \delta r_j + (1-\lambda _s)) $$
        $$ \hat {s}_j=s_j\cdot \text {min}(1.0, ~\lambda _s \delta s_j + (1-\lambda _s)) $$
        where $\lambda_s$ is a scaling factor for optimization stability [S1]. The transformed Gaussian is $G(x_j, \hat{r}_j, \hat{s}_j)$ [S1].

    *   **Camera Motion Blur:**
        The MLP outputs $M$ sets of offsets $(\delta x_j^{(i)}, \delta r_j^{(i)}, \delta s_j^{(i)})$ for $i=1 \dots M$ [S1]:
        $$ \{(\delta x_j^{(i)}, \delta r_j^{(i)}, \delta s_j^{(i)})\}_{i=1}^{M} = F_\theta \Big (\gamma (x_j), r_j, s_j, \gamma (v) \Big ) $$
        Compute transformed attributes for each set $i$ [S1]:
        $$ \hat{x}_j^{(i)} = x_j + \lambda_p \delta x_j^{(i)} $$
        $$ \hat{r}_j^{(i)} = r_j \cdot \text{min}(1.0, \lambda_s \delta r_j^{(i)} + (1-\lambda_s)) $$
        $$ \hat{s}_j^{(i)} = s_j \cdot \text{min}(1.0, \lambda_s \delta s_j^{(i)} + (1-\lambda_s)) $$
        where $\lambda_p$ scales the position offset [S1].
        Rasterize $M$ clean images $I_i$ from the $M$ sets of transformed Gaussians [S1]:
        $$ I_i = \texttt{Rasterize}(\{G(\hat{x}_j^{(i)}, \hat{r}_j^{(i)}, \hat{s}_j^{(i)})\}_{j=1}^{N_G}) $$
        Average them to simulate the blurred image [S1]:
        $$ I_b = \frac {1}{M}\sum _{i=1}^{M} I_i $$

3.  **Optimization:**
    Optimize the MLP parameters $\theta$ and the 3D Gaussian positions $x$ using the Adam optimizer [S1]. The loss is computed between the rendered image $I_b$ (or $I_i$ for defocus) and the ground truth blurry image [S1].

4.  **Inference:**
    The MLP $F_\theta$ is **not** activated. The scene is rendered using the original, untransformed Gaussians $G(x_j, r_j, s_j)$ [S1]. This ensures real-time rendering and sharp images [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(p, L=10):
    """
    Computes positional encoding for vector p.
    p: numpy array of shape (..., 3)
    L: number of frequencies
    Returns: numpy array of shape (..., 2*L*3)
    """
    # p is expected to be in range [-1, 1] or similar, scaled appropriately
    # Standard 3DGS positional encoding usually involves scaling
    # Formula: gamma(p) = (sin(2^k * pi * p), cos(2^k * pi * p)) for k=0..L-1
    freq_bands = 2.0 ** np.arange(L)
    # Reshape for broadcasting: p shape (..., 3), freq_bands shape (L,)
    # We need to expand dimensions to compute sin/cos for each band
    # Result shape: (..., L, 2, 3) -> flatten last dims to (..., 2*L*3)
    
    # Using einsum or manual expansion for clarity
    # p_expanded: (..., 1, 1, 3)
    # freq_bands: (L, 1, 1)
    # result: (..., L, 2, 3)
    
    # Note: The source formula is element-wise. 
    # Let's assume p is normalized.
    
    # Implementation detail: 
    # sin(2^k * pi * p) and cos(2^k * pi * p)
    
    # To avoid complex broadcasting in pure python-like pseudocode without libraries,
    # we describe the operation. In a real implementation, this is vectorized.
    
    # Placeholder for vectorized operation
    # freq = 2.0 ** np.arange(L)
    # angles = p[..., np.newaxis, np.newaxis, :] * freq[np.newaxis, np.newaxis, :, np.newaxis] * np.pi
    # enc = np.concatenate([np.sin(angles), np.cos(angles)], axis=-2)
    # return enc.reshape(p.shape[:-1] + (-1,))
    pass

def add_extra_points(P, Np, K, td):
    """
    Adds extra points to point cloud P to compensate for sparsity.
    P: list of (position, color) tuples
    Np: number of points to add
    K: number of neighbors
    td: distance threshold
    """
    # Get min/max of positions
    positions = np.array([p[0] for p in P])
    alpha = np.min(positions, axis=0)
    beta = np.max(positions, axis=0)
    
    # Sample Np points uniformly
    new_points = np.random.uniform(alpha, beta, size=(Np, 3))
    
    for p_pos in new_points:
        # Find K nearest neighbors
        # Calculate distances
        dists = np.linalg.norm(positions - p_pos, axis=1)
        k_indices = np.argsort(dists)[:K]
        k_neighbors = [P[i] for i in k_indices]
        
        # Filter by distance threshold
        valid_neighbors = []
        for i, neighbor in zip(k_indices, k_neighbors):
            if dists[i] < td:
                valid_neighbors.append(neighbor)
        
        if len(valid_neighbors) > 0:
            # Interpolate color
            # Linear interpolation based on distance
            # Simple average for pseudocode completeness
            avg_color = np.mean([n[1] for n in valid_neighbors], axis=0)
            P.append((p_pos, avg_color))
            
    return P

def deblur_3dgs_training_step(gaussians, blurry_image, camera_pose, blur_type='defocus'):
    """
    One training step.
    """
    # 1. Get MLP outputs
    # F_theta is the MLP
    # Inputs: gamma(x), r, s, gamma(v)
    
    # For each Gaussian j
    for j in range(len(gaussians)):
        x_j = gaussians[j].position
        r_j = gaussians[j].rotation
        s_j = gaussians[j].scale
        v = camera_pose.view_direction # Simplified
        
        # Positional Encoding
        gamma_x = positional_encoding(x_j)
        gamma_v = positional_encoding(v)
        
        # MLP Forward Pass
        if blur_type == 'defocus':
            delta_r, delta_s = F_theta(gamma_x, r_j, s_j, gamma_v)
            
            # Transform attributes
            # lambda_s is a hyperparameter
            lambda_s = 1e-2
            
            # min(1.0, lambda_s * delta + (1 - lambda_s))
            scale_r = np.minimum(1.0, lambda_s * delta_r + (1 - lambda_s))
            scale_s = np.minimum(1.0, lambda_s * delta_s + (1 - lambda_s))
            
            r_hat = r_j * scale_r
            s_hat = s_j * scale_s
            
            # Render from transformed Gaussians
            # Note: In defocus, we render from the single set of transformed Gaussians
            # The source says "G(xj, r_hat, s_hat)"
            rendered_image = rasterize(gaussians, r_hat=r_hat, s_hat=s_hat)
            
        elif blur_type == 'motion':
            M = 5 # Hyperparameter
            lambda_p = 1e-2
            lambda_s = 1e-2
            
            # MLP outputs M sets of deltas
            # F_theta outputs (delta_x, delta_r, delta_s) for each of M samples
            # This implies the MLP might have a 'sample' input or we sample M times
            # Source: "M sets of offsets... for i=1..M"
            # Usually, this is done by adding a noise vector or sampling index to MLP input
            # Or the MLP outputs a tensor of shape (M, ...). 
            # Source Eq 10: { (delta x, delta r, delta s) }_{i=1}^M = F_theta(...)
            # This suggests F_theta outputs a batch of M predictions.
            
            deltas = F_theta(gamma_x, r_j, s_j, gamma_v, M=M) # Returns M sets
            
            images = []
            for i in range(M):
                delta_x = deltas['x'][i]
                delta_r = deltas['r'][i]
                delta_s = deltas['s'][i]
                
                # Transform
                x_hat = x_j + lambda_p * delta_x
                
                scale_r = np.minimum(1.0, lambda_s * delta_r + (1 - lambda_s))
                scale_s = np.minimum(1.0, lambda_s * delta_s + (1 - lambda_s))
                
                r_hat = r_j * scale_r
                s_hat = s_j * scale_s
                
                # Render from this set
                img_i = rasterize(gaussians, x_hat=x_hat, r_hat=r_hat, s_hat=s_hat)
                images.append(img_i)
            
            rendered_image = np.mean(images, axis=0)
            
    # Compute Loss
    loss = compute_loss(rendered_image, blurry_image)
    
    # Backpropagate to update MLP params and Gaussian positions
    update_parameters(loss)
    
    return loss

def deblur_3dgs_inference(gaussians, camera_pose):
    """
    Inference: Render sharp image using original Gaussians.
    """
    # Do NOT use MLP
    # Render using original x, r, s
    image = rasterize(gaussians)
    return image
```

**Parameters:**
*   $\lambda_s$: Scaling factor for defocus blur deltas ($\delta r, \delta s$). Value: $10^{-2}$ [S1].
*   $\lambda_p$: Scaling factor for motion blur position offset ($\delta x$). Value: $10^{-2}$ [S1].
*   $N_p$: Number of additional points. Value: Up to 200,000, proportional to point cloud extent [S1].
*   $t_d$: Distance threshold for neighbor validity. Value: 2 [S1].
*   $K$: Number of neighbors for interpolation. Value: 4 [S1].
*   MLP Learning Rate: $10^{-3}$ [S1].
*   Position Learning Rate: $1.6 \times 10^{-3}$ [S1].
*   MLP Hidden Units: 64 [S1].
*   $M$: Number of sets for motion blur. Value: 5 [S1].
*   $N_{st}$: Iterations to start adding extra points. Value: 2,500 [S1].
*   Pruning threshold multiplier $w_p$: Value: 3 [S1].

**Edge Cases and Pitfalls:**
*   **Sparse Point Clouds:** Blurry inputs cause SfM to fail, leading to sparse point clouds, especially at far distances [S1]. The method compensates by adding points via KNN interpolation and pruning based on depth to retain far-plane Gaussians [S1].
*   **Inference Speed:** The MLP is strictly disabled during inference. Activating it would degrade performance and is not part of the standard 3D-GS pipeline [S1].
*   **Precision:** The `min` operation clips the scaling factors to 1.0 to prevent excessive dispersion [S1].

## Operating characteristics
*   **Time Complexity:** Training involves MLP forward/backward passes and rasterization of $M$ sets of Gaussians for motion blur, making it more expensive than standard 3D-GS training. Inference is identical to standard 3D-GS, offering real-time performance [S1].
*   **Memory Complexity:** Requires storing the MLP parameters and, for motion blur training, rendering $M$ images per step. Inference memory is standard 3D-GS [S1].
*   **Performance:** Achieves state-of-the-art PSNR and SSIM on real camera motion blur and defocus blur datasets [S1]. Rendering speed exceeds 800 FPS (e.g., 961 FPS on the 'Girl' scene) [S1].
*   **Strengths:** Real-time inference; handles both defocus and motion blur; works with sparse SfM point clouds from blurry images [S1].
*   **Limitations:** Requires training with the MLP; performance depends on the quality of the initial SfM point cloud (mitigated by point addition) [S1].

## Background
Deblurring 3D Gaussian Splatting was proposed by B. Lee et al. in 2024 [S1]. It is the first deblurring algorithm for 3D-GS, addressing the limitations of NeRF-based deblurring methods (like Deblur-NeRF, DP-NeRF, PDRF) which rely on volumetric rendering and are slow [S1]. It builds on the 3D-GS framework [S1].

## Sources
[S1] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
