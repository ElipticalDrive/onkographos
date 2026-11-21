# Forplane

Also called: Factorized Implicit Representation, Fast Orthogonal Plane Representation, Orthogonal Neural Plane

## Overview
Forplane is a neural implicit representation method designed for efficient 4D reconstruction of deformable scenes, specifically targeting surgical endoscopy videos [S1]. It solves the problem of reconstructing dynamic, deformable tissues from monocular or binocular video by factorizing the 4D volume into a static 3D field and a dynamic 3D field, both represented by orthogonal 2D neural planes [S1]. Unlike methods that treat time steps as independent static volumes or rely on displacement fields that struggle with topological changes, Forplane uses space-time planes to model arbitrary deformations while sharing scene priors across time [S1]. It distinguishes itself from its sibling [K-planes](3317-k-planes.md) by explicitly separating static and dynamic components into distinct plane sets (space vs. space-time) and from [Feature Grid as Radiance Field](5508-feature-grid-as-radiance-field.md) by its specific application to deformable tissue reconstruction with spatiotemporal importance sampling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of endoscopic images (monocular or binocular) at various time steps $\tau$.
*   **Output:** A continuous function predicting density $\sigma$ and color $\mathbf{c}$ for any 4D coordinate $(x, y, z, \tau)$, enabling volume rendering of the tissue surface.

**Data Structures:**
The representation consists of two fields: a static field $V_{s3d}$ and a dynamic field $V_{d3d}$. Both use multi-resolution planes [S1].

1.  **Static Field $V_{s3d}$:** Composed of three orthogonal space planes: $G_{XY}, G_{YZ}, G_{XZ}$.
    *   Layout: Each plane is a 3D tensor of size $N \times N \times D$.
    *   $N$: Spatial resolution.
    *   $D$: Feature size (dimensionality of the feature vector).
2.  **Dynamic Field $V_{d3d}$:** Composed of three orthogonal space-time planes: $G_{TX}, G_{TY}, G_{TZ}$.
    *   Layout: Each plane is a 3D tensor of size $N \times M \times D$.
    *   $N$: Spatial resolution.
    *   $M$: Temporal resolution.
    *   $D$: Feature size.

**Algorithm Steps:**

1.  **Ray Casting:** For a target tissue pixel $p_{ij}$ at time step $\tau$, cast a ray $\mathbf{r}(t)$ from the camera origin $\mathbf{x}_o$ through the pixel.
2.  **Spatiotemporal Sampling:** Sample points along the ray to obtain 4D coordinates $\mathbf{q} = (x, y, z, \tau)$ [S1]. *Note: The method employs a spatiotemporal importance sampling strategy to prioritize pixels occluded by tools or exhibiting extensive motion, and to reduce redundant training on stationary tissues [S1].*
3.  **Feature Query:** For each sampled point $\mathbf{q}=(x,y,z,\tau)$, query features from the six planes using bilinear interpolation $\mathcal{B}$:
    *   Static features: $\mathbf{f}_s = \mathcal{B}(G_{XY}, x, y) \odot \mathcal{B}(G_{YZ}, y, z) \odot \mathcal{B}(G_{XZ}, x, z)$
    *   Dynamic features: $\mathbf{f}_d = \mathcal{B}(G_{TX}, \tau, x) \odot \mathcal{B}(G_{TY}, \tau, y) \odot \mathcal{B}(G_{TZ}, \tau, z)$
    *   *Note: The source formula (4) in [S1] lists $G_{YT}$ and $G_{ZT}$ but the text defines the dynamic planes as $G_{TX}, G_{TY}, G_{TZ}$. The standard orthogonal set for space-time involves time paired with each spatial axis. We assume the set $\{G_{TX}, G_{TY}, G_{TZ}\}$ is the correct implementation based on the textual description of "space-time planes".*
    *   Fused feature vector $\mathbf{v}$ is obtained by combining static and dynamic features. The source states $\mathbf{v}$ is concatenated with positional encoded coordinates and time embeddings [S1].
4.  **Decoding:** Pass the concatenated vector through a lightweight MLP $\theta$ to predict density $\sigma$ and color $\mathbf{c}$ [S1].
    *   MLP Architecture: 2 fully connected layers, each with 64 channels [S1].
5.  **Volume Rendering:** Compute the predicted color $\hat{C}(\mathbf{r})$ for the ray using standard volume rendering integration:
    $$ \hat{C}(\mathbf{r}) = \int_{t_n}^{t_f} w(t) \mathbf{c}(\mathbf{r}(t)) dt $$
    where $w(t)$ is the accumulated weight along the ray [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def query_forplane(q_xyz, tau, G_XY, G_YZ, G_XZ, G_TX, G_TY, G_TZ, MLP):
    """
    Query the Forplane representation for a single 4D point.
    
    Args:
        q_xyz: numpy array of shape (3,), coordinates (x, y, z).
        tau: float, time coordinate.
        G_XY, G_YZ, G_XZ: numpy arrays of shape (N, N, D), static space planes.
        G_TX, G_TY, G_TZ: numpy arrays of shape (N, M, D), dynamic space-time planes.
        MLP: function or object representing the 2-layer, 64-channel MLP.
        
    Returns:
        sigma: float, predicted density.
        c: numpy array of shape (3,), predicted color.
    """
    x, y, z = q_xyz
    
    # Bilinear interpolation helper
    def bilinear_interp(grid, u, v):
        # grid shape: (H, W, D)
        # u, v are continuous coordinates in [0, H-1] and [0, W-1]
        # Implementation of bilinear interpolation is assumed standard
        pass 

    # Query Static Field
    # G_XY: spatial x, y
    feat_xy = bilinear_interp(G_XY, x, y)
    # G_YZ: spatial y, z
    feat_yz = bilinear_interp(G_YZ, y, z)
    # G_XZ: spatial x, z
    feat_xz = bilinear_interp(G_XZ, x, z)
    
    # Element-wise multiplication for static features
    feat_static = feat_xy * feat_yz * feat_xz
    
    # Query Dynamic Field
    # G_TX: time tau, spatial x
    feat_tx = bilinear_interp(G_TX, tau, x)
    # G_TY: time tau, spatial y
    feat_ty = bilinear_interp(G_TY, tau, y)
    # G_TZ: time tau, spatial z
    feat_tz = bilinear_interp(G_TZ, tau, z)
    
    # Element-wise multiplication for dynamic features
    feat_dynamic = feat_tx * feat_ty * feat_tz
    
    # Combine features
    # Source states: "The fused feature vector v is concatenated with positional encoded coordinate and time embeddings"
    # [G] Positional encoding function PE is assumed standard (e.g., from NeRF)
    pos_enc_xyz = positional_encoding(x, y, z)
    pos_enc_tau = positional_encoding(tau)
    
    input_vector = np.concatenate([feat_static, feat_dynamic, pos_enc_xyz, pos_enc_tau])
    
    # Decode
    output = MLP(input_vector)
    
    # [G] Assuming MLP outputs [sigma, r, g, b] or similar. 
    # Source says "predicts color c and density sigma".
    # We assume the last layer produces these values.
    sigma = output[0]
    c = output[1:4]
    
    return sigma, c

def render_forplane(ray_origin, ray_direction, tau, G_XY, G_YZ, G_XZ, G_TX, G_TY, G_TZ, MLP, t_near, t_far, num_samples):
    """
    Render a single ray using Forplane.
    """
    # 1. Sample points along the ray
    t_vals = np.linspace(t_near, t_far, num_samples)
    points = ray_origin + ray_direction * t_vals[:, np.newaxis]
    
    colors = []
    weights = []
    alpha_accum = 0.0
    
    for i, point in enumerate(points):
        sigma, c = query_forplane(point, tau, G_XY, G_YZ, G_XZ, G_TX, G_TY, G_TZ, MLP)
        
        # [G] Standard volume rendering step
        # alpha = 1 - exp(-sigma * delta_t)
        delta_t = t_vals[i+1] - t_vals[i] if i < num_samples - 1 else t_far - t_vals[i]
        alpha = 1.0 - np.exp(-sigma * delta_t)
        
        transmittance = np.exp(-alpha_accum * delta_t) # Approximation, exact accumulation needed
        # Correct accumulation:
        # w_i = alpha_i * exp(-sum_{j<i} sigma_j * delta_t_j)
        
        # Simplified accumulation for pseudocode clarity
        alpha_accum += sigma * delta_t
        
        # [G] Standard volume rendering formula
        # C = sum(T_i * alpha_i * c_i)
        # T_i = exp(-sum_{j=0}^{i-1} sigma_j * delta_t_j)
        
        # Note: Full implementation requires storing sigma history for T_i calculation
        pass 
        
    # [G] Return final color
    return 0.0, 0.0, 0.0
```

**Parameters:**
*   $N$: Spatial resolution of planes.
*   $M$: Temporal resolution of dynamic planes.
*   $D$: Feature size (dimensionality).
*   MLP: 2 fully connected layers, 64 channels each [S1].

**Edge Cases and Pitfalls:**
*   **Tool Occlusion:** Tools occlude tissues, leading to infrequent representation in training sets and inaccurate reconstruction of occluded areas [S1].
*   **Stationary Tissues:** Repeated training on stationary tissues yields minimal impact on convergence and reduces efficiency [S1].
*   **Topology Changes:** Unlike displacement fields, Forplane's dynamic field can model complex deformations and topology changes, which is a strength, but requires sufficient dynamic plane capacity [S1].

## Subtopics

### [HUGS](17288-hugs.md)
A method for animating virtual human avatars using 3D Gaussian Splatting. It builds on the tri-plane representation to factorize canonical space, enabling reconstruction from monocular video. It distinguishes itself by bypassing inverse skinning, allowing for loose clothing and real-time rendering.

## Operating characteristics
*   **Complexity:** Reduces computational cost from $O(N^4)$ (traversing all spacetime positions) to $O(N^2)$ (querying planes) [S1].
*   **Performance:** Enables faster optimization and inference compared to treating time steps as independent static volumes or using complex displacement fields [S1].
*   **Strengths:**
    *   Efficient representation of 4D deformable scenes [S1].
    *   Shares learned scene priors across time steps via the static field [S1].
    *   Models arbitrary deformations without topology constraints [S1].
    *   Lightweight MLP (2 layers, 64 channels) reduces burden compared to methods like EndoNeRF (8 layers, 256 channels) [S1].
*   **Limitations:**
    *   Struggles with accurately reconstructing tissues occluded by tools due to lack of training data [S1].
    *   Efficiency can be degraded if not using importance sampling for stationary regions [S1].
*   **When to use:** Suitable for reconstructing deformable tissues from endoscopy videos where computational efficiency and handling of deformation are critical.
*   **When not to use:** Not suitable for scenes with heavy tool occlusion where accurate reconstruction of occluded regions is required without additional strategies.

## Background
Forplane was introduced in the paper "Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane" [S1]. It addresses challenges in surgical scene reconstruction, specifically the difficulty of modeling deformable tissues from monocular or binocular endoscopy videos. It builds on the principles of Planar Factorization [17382-planar-factorization.md] and improves upon previous methods like EndoNeRF [S1] by factorizing the 4D volume into static and dynamic fields using orthogonal planes.

## Sources
[S1] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
