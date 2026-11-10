# Forplane

Also called: Factorized Implicit Representation, Fast Orthogonal Plane Representation, Orthogonal Neural Plane

## Overview
Forplane is a neural implicit representation method designed for efficient 4D (spatiotemporal) reconstruction of deformable scenes, specifically targeting surgical video where tools cause occlusions and tissues deform [S1]. It solves the high computational cost of 4D volume traversal by factorizing the 4D volume into a static 3D field and a dynamic 3D field, both represented by 2D orthogonal planes [S1]. Unlike methods that treat time steps as independent static volumes or rely on displacement fields, Forplane separates static scene priors from dynamic deformations, reducing computational complexity from $O(N^4)$ to $O(N^2)$ while maintaining reconstruction quality [S1]. It builds on [Neural and primitive-based representations](10-neural-and-primitive-based-representations.md) by using a factorized grid structure instead of a monolithic MLP or voxel grid.

## How it works
**Inputs and Outputs:**
*   **Input:** Monocular or binocular endoscopy video frames (images) at various time steps $\tau$. Camera intrinsics and extrinsics are required for ray casting.
*   **Output:** Predicted color $\hat{C}$ and density $\sigma$ for any 3D spatial coordinate $\mathbf{x}$ at a specific time $\tau$, enabling volume rendering of the scene.

**Data Structures:**
The representation consists of two sets of 2D feature planes, each with dimensions $N \times N \times D$ (space) or $N \times M \times D$ (space-time), where $N$ is spatial resolution, $M$ is temporal resolution, and $D$ is the feature dimension.
1.  **Static Field ($V_s^{3d}$):** Composed of three orthogonal space planes: $G_{XY}, G_{YZ}, G_{XZ}$. These capture the static geometry and appearance of the scene.
2.  **Dynamic Field ($V_d^{3d}$):** Composed of three orthogonal space-time planes: $G_{TX}, G_{TY}, G_{TZ}$. These capture time-dependent deformations and appearance changes.
3.  **MLP ($\theta$):** A lightweight network with 2 fully connected layers, each having 64 channels [S1].

**Algorithm Steps:**
1.  **Ray Casting:** For a target tissue pixel $p_{ij}$ at time step $\tau$, cast a ray $\mathbf{r}(t)$ from the camera origin $\mathbf{x}_o$ through the pixel.
2.  **Spatiotemporal Sampling:** Sample points along the ray to obtain 4D coordinates $(x, y, z, \tau)$. The method employs **spatiotemporal importance sampling** to prioritize pixels that are occluded by tools or exhibit significant motion, addressing the issue of infrequent representation of occluded tissues and redundant training on stationary tissues [S1].
3.  **Feature Querying:** For each sampled point $\mathbf{P}(x, y, z, \tau)$, query features from the six planes using bilinear interpolation $\mathcal{B}$.
    *   Static features are queried from $G_{XY}, G_{YZ}, G_{XZ}$ using spatial coordinates.
    *   Dynamic features are queried from $G_{TX}, G_{TY}, G_{TZ}$ using spatial and temporal coordinates.
    *   The fused feature vector $\mathbf{v}$ is computed by element-wise multiplication ($\odot$) of the interpolated features:
        $$
        \mathbf{v}(x, y, z, \tau) = \mathcal{B}(G_{XY}, x, y) \odot \mathcal{B}(G_{YZ}, y, z) \odot \mathcal{B}(G_{XZ}, x, z) \odot \mathcal{B}(G_{TX}, x, \tau) \odot \mathcal{B}(G_{TY}, y, \tau) \odot \mathcal{B}(G_{TZ}, z, \tau)
        $$
        Note: The source formula (Eq. 4) explicitly lists $\mathcal{B}(G_{XY}, x, y)$, $\mathcal{B}(G_{YZ}, y, z)$, $\mathcal{B}(G_{YT}, y, \tau)$, and $\mathcal{B}(G_{ZT}, z, \tau)$ [S1]. However, the text defines the dynamic planes as $G_{TX}, G_{TY}, G_{TZ}$ and static as $G_{XY}, G_{YZ}, G_{XZ}$. The formula in the source appears to have a typo ($G_{YT}, G_{ZT}$ vs $G_{TY}, G_{TZ}$) or implies a specific ordering. Given the definition "space-time planes (i.e., $G_{TX}, G_{TY}$ and $G_{TZ}$)", the standard interpretation for orthogonal planes is to project onto the relevant 2D subspace. For $G_{TX}$, the coordinates are $x$ and $\tau$. For $G_{TY}$, $y$ and $\tau$. For $G_{TZ}$, $z$ and $\tau$. The static planes use spatial coordinates. The source formula snippet provided is: $\mathcal{B}(G_{XY}, x, y) \odot \mathcal{B}(G_{YZ}, y, z) \cdots \mathcal{B}(G_{YT}, y, \tau) \odot \mathcal{B}(G_{ZT}, z, \tau)$. We will use the explicit variables from the text definition for correctness in implementation, assuming the source formula had typos in the plane indices ($YT \to TY, ZT \to TZ$) or omitted the static $XZ$ and dynamic $TX$ terms in the snippet. A complete query requires all 6 planes.
4.  **MLP Prediction:** Concatenate the fused feature vector $\mathbf{v}$ with positional encoded coordinates and time embeddings. Pass this combined vector through the MLP $\theta$ to predict density $\sigma$ and color $\mathbf{c}$.
5.  **Volume Rendering:** Compute the predicted color $\hat{C}(\mathbf{r})$ for the ray using standard volume rendering integration:
    $$
    \hat{C}(\mathbf{r}) = \int_{t_n}^{t_f} w(t) \mathbf{c}(\mathbf{r}(t), \mathbf{d}) dt
    $$
    where $w(t)$ is the weight determined by density and transmittance, $t_n$ and $t_f$ are near and far clipping planes, and $\mathbf{d}$ is the viewing direction [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def query_forplane(G_static, G_dynamic, x, y, z, tau, D):
    """
    Query features from Forplane planes.
    G_static: dict with keys 'XY', 'YZ', 'XZ', each shape (N, N, D)
    G_dynamic: dict with keys 'TX', 'TY', 'TZ', each shape (N, M, D)
    x, y, z: spatial coordinates (normalized 0-1 or scaled to N)
    tau: time coordinate (normalized 0-1 or scaled to M)
    D: feature dimension
    """
    # Bilinear interpolation helper
    def bilinear_interp(plane, u, v):
        # u, v are coordinates in [0, N] or [0, M]
        # Clamp to boundaries
        u = np.clip(u, 0, plane.shape[0] - 1)
        v = np.clip(v, 0, plane.shape[1] - 1)
        
        # Floor and ceil indices
        u0 = np.floor(u).astype(int)
        v0 = np.floor(v).astype(int)
        u1 = u0 + 1
        v1 = v0 + 1
        
        # Handle boundary cases
        u1 = np.minimum(u1, plane.shape[0] - 1)
        v1 = np.minimum(v1, plane.shape[1] - 1)
        
        # Interpolation weights
        du = u - u0
        dv = v - v0
        
        # Get corner values
        q00 = plane[u0, v0, :]
        q10 = plane[u1, v0, :]
        q01 = plane[u0, v1, :]
        q11 = plane[u1, v1, :]
        
        # Bilinear interpolation
        top = q00 * (1 - du) + q10 * du
        bottom = q01 * (1 - du) + q11 * du
        return top * (1 - dv) + bottom * dv

    # Query Static Planes
    v_xy = bilinear_interp(G_static['XY'], x, y)
    v_yz = bilinear_interp(G_static['YZ'], y, z)
    v_xz = bilinear_interp(G_static['XZ'], x, z)
    
    # Query Dynamic Planes
    # Note: G_TX uses x and tau, G_TY uses y and tau, G_TZ uses z and tau
    v_tx = bilinear_interp(G_dynamic['TX'], x, tau)
    v_ty = bilinear_interp(G_dynamic['TY'], y, tau)
    v_tz = bilinear_interp(G_dynamic['TZ'], z, tau)
    
    # Element-wise multiplication of all 6 features
    v = v_xy * v_yz * v_xz * v_tx * v_ty * v_tz
    
    return v

def predict_color_density(v, pos_enc_x, pos_enc_y, pos_enc_z, pos_enc_tau, MLP):
    """
    Pass fused feature and embeddings through MLP.
    MLP: 2 FC layers, 64 channels each.
    """
    # Concatenate features and embeddings
    input_vec = np.concatenate([v, pos_enc_x, pos_enc_y, pos_enc_z, pos_enc_tau])
    
    # MLP Forward Pass
    # Layer 1: Linear + ReLU
    h = np.maximum(0, np.dot(input_vec, MLP.W1) + MLP.b1)
    # Layer 2: Linear (output sigma and c)
    output = np.dot(h, MLP.W2) + MLP.b2
    
    # Split output into density (sigma) and color (c)
    # Assuming last D channels are color, and 1 channel is density, or similar layout.
    # The source says "predicts color c and density sigma". 
    # Exact split depends on implementation, but typically sigma is scalar, c is vector.
    sigma = output[0] # Example assumption
    c = output[1:]    # Example assumption
    
    return sigma, c

def render_ray(G_static, G_dynamic, ray_origin, ray_direction, tau, MLP, near, far, num_samples):
    """
    Render a single ray using volume rendering.
    """
    # Sample points along ray
    t_vals = np.linspace(near, far, num_samples)
    points = ray_origin + ray_direction * t_vals[:, np.newaxis]
    
    # Extract x, y, z
    x, y, z = points[:, 0], points[:, 1], points[:, 2]
    
    # Query features
    features = query_forplane(G_static, G_dynamic, x, y, z, tau, D=64)
    
    # Predict sigma and c
    # Note: Positional encoding should be applied to x,y,z,tau before concatenation
    # This is a simplified placeholder for the MLP call
    sigmas, colors = predict_color_density(features, x, y, z, tau, MLP)
    
    # Volume Rendering Integration (Standard Alpha Compositing)
    # Convert sigma to alpha: alpha = 1 - exp(-sigma * delta_t)
    deltas = t_vals[1:] - t_vals[:-1]
    deltas = np.append(deltas, 1e-10) # Handle last point
    
    alphas = 1.0 - np.exp(-sigmas * deltas)
    transmittance = np.exp(-np.cumsum(sigmas * deltas))
    weights = alphas * transmittance
    
    # Weighted sum of colors
    color = np.sum(weights[:, np.newaxis] * colors, axis=0)
    
    return color
```

**Parameters:**
*   **MLP Architecture:** 2 fully connected layers, 64 channels each [S1].
*   **Plane Dimensions:** $N \times N \times D$ for static, $N \times M \times D$ for dynamic. $N$ is spatial resolution, $M$ is temporal resolution, $D$ is feature size.
*   **Multi-resolution:** Both static and dynamic fields use multi-resolution planes, inspired by hybrid representations [S1].

**Edge Cases and Pitfalls:**
*   **Tool Occlusion:** Tools occlude tissues, leading to infrequent representation in training data and inaccurate reconstruction of occluded areas [S1]. The method uses spatiotemporal importance sampling to mitigate this by prioritizing these difficult pixels.
*   **Stationary Tissues:** Repeated training on stationary tissues yields minimal convergence impact and reduces efficiency [S1]. Importance sampling also addresses this by reducing redundant updates.
*   **Precision:** Bilinear interpolation is used for feature querying. Ensure coordinate normalization matches the plane dimensions.

## Operating characteristics
*   **Complexity:** Reduces computational cost from $O(N^4)$ (traversing all spacetime positions) to $O(N^2)$ (querying 2D planes) [S1].
*   **Memory:** Memory usage is determined by the size of the 6 planes ($6 \times N^2 \times D$ or $6 \times N \times M \times D$) plus the small MLP. It is constant with respect to the continuous spatial resolution, allowing infinite resolution reconstruction [S1].
*   **Performance:** Faster convergence and inference compared to methods treating time steps independently or using displacement fields [S1].
*   **Strengths:** Efficient handling of 4D data, ability to model arbitrary deformations without topology changes, sharing of static scene priors across time steps [S1].
*   **Limitations:** Relies on the assumption that deformations can be factorized into static and dynamic components. Performance may degrade if the factorization assumption is violated or if occlusions are severe and not well-handled by the sampling strategy.

## Background
Forplane was introduced in "Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane" (2024) [S1]. It addresses challenges in surgical scene reconstruction, specifically deformable tissue and tool occlusion. It builds on techniques from NeRF and EndoNeRF [S1]. The method is designed for medical imaging applications, particularly endoscopy.

## Sources
[S1] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
