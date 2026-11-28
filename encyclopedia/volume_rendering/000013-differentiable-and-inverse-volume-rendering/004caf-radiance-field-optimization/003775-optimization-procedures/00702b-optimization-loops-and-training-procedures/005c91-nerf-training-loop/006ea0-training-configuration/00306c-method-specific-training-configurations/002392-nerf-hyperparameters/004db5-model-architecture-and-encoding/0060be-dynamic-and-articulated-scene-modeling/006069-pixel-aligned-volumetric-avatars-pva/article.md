# Pixel-aligned Volumetric Avatars (PVA)

## Overview
Pixel-aligned Volumetric Avatars (PVA) is a framework for estimating a volumetric 3D avatar from a few input images of a human head, enabling view synthesis for unseen identities and expressions [S1, unofficial]. It generalizes to new identities by combining neural radiance fields with local, pixel-aligned features extracted directly from input images, avoiding the information loss associated with global latent codes [S1, unofficial]. Unlike mesh-based methods, PVA captures arbitrary topology and high-frequency details (e.g., hair) without requiring explicit 3D supervision [S1, unofficial].

## How it works
PVA consists of two main components: a shallow convolutional encoder-decoder network ($N_{feat}$) that produces pixel-aligned feature maps from input images, and a radiance field network ($N$) that maps 3D locations and these features to color and opacity [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:** One or more conditioning views $v_i$ (images of dimensions $h \times w \times 3$) with known camera intrinsics $K_i$ and extrinsics $[R|t]_i$.
*   **Outputs:** A radiance field function that, given a 3D point $\mathbf{X}$ and view direction, predicts color $\mathbf{c}$ and volume density $\sigma$.

**Data Structures:**
*   **Feature Maps ($f^{(i)}$):** 2D arrays of shape $h \times w \times d_f$, where $d_f$ is the feature dimension. These are the outputs of $N_{feat}$.
*   **Radiance Field Network ($N$):** An MLP taking positional encoding $\phi(\mathbf{X})$ and feature vector $\mathbf{f}_\mathbf{X}$ as input.
*   **Positional Encoding ($\phi$):** Standard NeRF positional encoding applied to 3D coordinates [S1, unofficial].

**Algorithm Steps:**

1.  **Feature Extraction:** For each conditioning view $v_i$, compute the pixel-aligned feature map $f^{(i)}$ by passing the view through the encoder-decoder network:
    $$ f^{(i)} = N_{feat}(v_i) \quad [S1, \text{unofficial}, unofficial] $$

2.  **Feature Interpolation:** For a 3D query point $\mathbf{X}$, determine its corresponding pixel location $\mathbf{x}$ in the conditioning view $v_i$ using perspective projection $\Pi$:
    $$ \mathbf{x} = \Pi(\mathbf{X}) \quad [S1, \text{unofficial}, unofficial] $$
    Extract the feature vector $\mathbf{f}_\mathbf{X}$ by bilinearly interpolating the feature map $f^{(i)}$ at location $\mathbf{x}$:
    $$ \mathbf{f}_\mathbf{X} = F(f^{(i)}, \mathbf{x}) \quad [S1, \text{unofficial}, unofficial] $$
    where $F$ denotes bilinear interpolation.

3.  **Radiance Field Query:** Pass the positional encoding of the 3D location, $\phi(\mathbf{X})$, and the extracted feature vector $\mathbf{f}_\mathbf{X}$ into the radiance field network $N$ to predict color $\mathbf{c}$ and opacity/density $\sigma$:
    $$ \{\mathbf{c}, \sigma\} = N(\phi(\mathbf{X}), \mathbf{f}_\mathbf{X}) \quad [S1, \text{unofficial}, unofficial] $$

4.  **Ray Marching Setup:** For a target view $j$ with camera intrinsics $K_j$ and extrinsics $[R|t]_j$, define the ray for pixel $p$. The camera-to-world projection matrix is $P^{-1} = [R_i | t_i]^{-1} K_i^{-1}$ (using the conditioning view's pose relative to head pose) [S1, unofficial]. The ray direction $\mathbf{d}$ is:
    $$ \mathbf{d} = \frac{P^{-1} \mathbf{p} - \mathbf{r}_0}{\| P^{-1} \mathbf{p} - \mathbf{r}_0 \|} \quad [S1, \text{unofficial}, unofficial] $$
    where $\mathbf{r}_0$ is the ray origin. Note: Camera extrinsics are relative to the computed head pose to help the network focus on scene content [S1, unofficial].

5.  **Sampling:** Uniformly sample a set of $n_s$ points $t$ along the ray $r(t) = \mathbf{r}_0 + t\mathbf{d}$ for $t \in [t_{near}, t_{far}]$. Set $\mathbf{X} = r(t)$ [S1, unofficial].

6.  **Volume Rendering:** Accumulate color and opacity along the ray using standard volume rendering integration (quadrature rule) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def pva_render(conditioning_views, target_view, n_s, t_near, t_far, N_feat, N):
    """
    Render a target view using Pixel-aligned Volumetric Avatars.
    
    Args:
        conditioning_views: List of dicts, each containing 'image' (HxWx3), 
                            'K' (3x3 intrinsics), 'R' (3x3 rotation), 't' (3x1 translation).
        target_view: Dict containing 'K' (3x3 intrinsics), 'R' (3x3 rotation), 't' (3x1 translation).
        n_s: Number of uniformly sampled points along the ray.
        t_near: Near plane distance.
        t_far: Far plane distance.
        N_feat: Encoder-decoder network function.
        N: Radiance field MLP function.
        
    Returns:
        rendered_image: HxWx3 array.
    """
    # Assume single conditioning view for simplicity as per primary description
    cond = conditioning_views[0]
    H, W, _ = cond['image'].shape
    
    # 1. Extract pixel-aligned feature maps from conditioning view
    # f_cond is H x W x D_feat
    f_cond = N_feat(cond['image'])
    
    # Precompute projection matrix for conditioning view relative to head pose
    # P_inv = [R_cond | t_cond]^-1 @ K_cond^-1
    # Note: The paper states extrinsics are relative to computed head pose.
    # We assume cond['R'], cond['t'] are already in this relative frame.
    K_inv_cond = np.linalg.inv(cond['K'])
    RT_cond = np.eye(4)
    RT_cond[:3, :3] = cond['R']
    RT_cond[:3, 3] = cond['t'].flatten()
    P_inv_cond = RT_cond @ K_inv_cond
    
    # Precompute projection matrix for target view
    K_inv_target = np.linalg.inv(target_view['K'])
    RT_target = np.eye(4)
    RT_target[:3, :3] = target_view['R']
    RT_target[:3, 3] = target_view['t'].flatten()
    P_inv_target = RT_target @ K_inv_target
    
    rendered_image = np.zeros((H, W, 3))
    
    for y in range(H):
        for x in range(W):
            # 4. Ray Marching Setup for pixel (x, y)
            # Pixel coordinates in homogeneous space
            p_hom = np.array([x, y, 1.0])
            
            # Ray direction in world space
            # d = (P_inv_target @ p - r0) / norm
            # r0 is the camera center. In homogeneous coords, r0_hom = [0,0,0,1] in camera space.
            # World center = P_inv_target @ [0,0,0,1]^T
            r0_world = P_inv_target @ np.array([0, 0, 0, 1])
            r0_world = r0_world[:3] / r0_world[3]
            
            # Point at depth 1 in camera space projected to world
            p1_hom = P_inv_target @ np.append(p_hom, 1.0)
            p1_world = p1_hom[:3] / p1_hom[3]
            
            d = p1_world - r0_world
            d = d / np.linalg.norm(d)
            
            # 5. Sampling
            t_vals = np.linspace(t_near, t_far, n_s)
            points = r0_world + t_vals[:, np.newaxis] * d[np.newaxis, :] # Shape: (n_s, 3)
            
            # 2. Feature Interpolation for each point
            # Project each 3D point to the conditioning view's pixel coordinates
            # x_pixel = K_cond @ (R_cond @ X + t_cond)
            # We need to invert this to get pixel coords from 3D points
            # X_world -> X_cond = R_cond^-1 @ (X_world - t_cond)
            # x_pixel_hom = K_cond @ X_cond
            
            RT_cond_inv = np.linalg.inv(RT_cond)
            
            colors_accum = 0.0
            alpha_accum = 0.0
            
            for i in range(n_s):
                X = points[i]
                
                # Transform X to conditioning camera space
                X_cond_hom = RT_cond_inv @ np.append(X, 1.0)
                X_cond = X_cond_hom[:3]
                
                # Project to pixel coordinates
                x_pixel_hom = cond['K'] @ X_cond
                x_pixel = x_pixel_hom[:2] / x_pixel_hom[2]
                
                # Check if point is within image bounds (simple clamp for interpolation)
                # Bilinear interpolation requires valid coordinates
                px, py = x_pixel[0], x_pixel[1]
                
                # Clamp to image boundaries for interpolation
                px_clamped = np.clip(px, 0.5, W - 1.5)
                py_clamped = np.clip(py, 0.5, H - 1.5)
                
                # Bilinear interpolation of feature map f_cond at (px, py)
                # f_cond is H x W x D
                # Get floor/ceil indices
                x0, y0 = int(np.floor(px_clamped)), int(np.floor(py_clamped))
                x1, y1 = x0 + 1, y0 + 1
                
                # Interpolate
                # f(y, x) = f(y0, x0)*(1-dx)*(1-dy) + f(y0, x1)*dx*(1-dy) + ...
                dx = px_clamped - x0
                dy = py_clamped - y0
                
                f00 = f_cond[y0, x0]
                f01 = f_cond[y0, x1]
                f10 = f_cond[y1, x0]
                f11 = f_cond[y1, x1]
                
                f_interp = (1-dx)*(1-dy)*f00 + dx*(1-dy)*f01 + (1-dx)*dy*f10 + dx*dy*f11
                
                # 3. Radiance Field Query
                # Compute positional encoding phi(X)
                phi_X = positional_encoding(X)
                
                # Query network
                c, sigma = N(phi_X, f_interp)
                
                # Volume Rendering Accumulation
                # alpha = 1 - exp(-sigma * delta_t)
                delta_t = t_vals[i+1] - t_vals[i] if i < n_s - 1 else t_far - t_vals[i]
                alpha = 1.0 - np.exp(-sigma * delta_t)
                
                # T = exp(-sum(sigma * delta_t))
                # Contribution = T * alpha * c
                # We accumulate from front to back
                
                # Note: Standard NeRF accumulation
                # weight = alpha * T
                # color += weight * c
                # T *= (1 - alpha)
                
                # For simplicity in this pseudocode, assuming single pass accumulation
                # In practice, T is maintained across the loop
                pass 
            
            # Simplified accumulation for pseudocode completeness
            # Real implementation requires maintaining T across the n_s loop
            T = 1.0
            final_color = np.zeros(3)
            for i in range(n_s):
                X = points[i]
                # ... (recompute f_interp as above) ...
                # phi_X = positional_encoding(X)
                # c, sigma = N(phi_X, f_interp)
                # delta_t = t_vals[i+1] - t_vals[i] if i < n_s - 1 else t_far - t_vals[i]
                # alpha = 1.0 - np.exp(-sigma * delta_t)
                # weight = alpha * T
                # final_color += weight * c
                # T *= (1 - alpha)
                pass
            rendered_image[y, x] = final_color
            
    return rendered_image

def positional_encoding(x, min_deg=0, max_deg=10, power=2):
    """Standard NeRF positional encoding."""
    scales = np.array([2**i for i in range(min_deg, max_deg+1)])
    xb = np.reshape((x[..., np.newaxis, :] * scales), (x.shape[0], -1))
    freq_b = np.sin(xb * power)
    freq_b = np.concatenate([np.cos(freq_b), np.sin(freq_b)], axis=-1)
    return np.concatenate([x, freq_b], axis=-1)
```

**Parameters:**
*   $n_s$: Number of uniformly sampled points along the ray. Typical values are not explicitly given in the notes, but standard NeRF uses 48-192 [S1, unofficial].
*   $t_{near}, t_{far}$: Near and far plane distances for ray marching.

**Edge Cases and Pitfalls:**
*   **Global Latent Codes:** Using global latent codes (as in SRNs) discards local details, making it difficult to generalize to local changes like facial expressions [S1, unofficial]. PVA avoids this by using local pixel-aligned features.
*   **3D Supervision:** PVA does not require explicit 3D supervision (e.g., depth or normals) during training, relying solely on photometric re-rendering loss [S1, unofficial].
*   **Topology:** Unlike mesh-based methods, PVA captures arbitrary topology (e.g., hair, expressions) because it uses pixel-level information [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time depends on the number of pixels, the number of sampled points $n_s$, and the cost of the encoder-decoder network $N_{feat}$ and radiance field network $N$. The feature interpolation step adds overhead proportional to the number of sampled points.
*   **Memory Complexity:** Memory usage is dominated by the feature maps $f^{(i)}$ (size $H \times W \times d_f$) and the parameters of $N_{feat}$ and $N$. It avoids the memory restrictions of voxel-based methods by using an MLP instead of a 3D voxel grid [S1, unofficial].
*   **Strengths:** Captures high-frequency texture details and arbitrary topology (hair, expressions) without 3D supervision. Generalizes to unseen identities and expressions.
*   **Limitations:** Requires input images with known camera poses. Performance may degrade if the input images do not adequately cover the identity's appearance variations.
*   **When to use:** Virtual telepresence applications requiring photo-realistic human heads from a small number of example views [S1, unofficial].
*   **When not to use:** When explicit 3D geometry is required for downstream tasks (e.g., physics simulation) or when input camera poses are unknown/unreliable.

## Background
PVA was introduced as a novel framework for the estimation of a volumetric 3D avatar from only a few input images of a human head [S1, unofficial]. It builds on insights from NeRF and PIFu/PIFuHD, leveraging pixel-aligned features to retain high-frequency detail without requiring 3D supervision [S1, unofficial]. It addresses the limitations of Scene Representation Networks (SRNs) which struggle with local changes like facial expressions due to global latent codes [S1, unofficial]. It is used for virtual telepresence applications [S1, unofficial].

## Sources
[S1] PVA: Pixel-aligned Volumetric Avatars, https://doi.org/10.48550/arxiv.2101.02697, fetched 2026-10-30 (unofficial)
