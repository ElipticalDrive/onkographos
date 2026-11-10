# Quadri-Plane Radiance Fields

Also called: 4-Plane Representation, Quadri-Plane Model

## Overview
Quadri-Plane Radiance Fields represent a 3D scene using four multi-channel 2D feature planes coupled with a lightweight decoder MLP, designed to enable computationally efficient volumetric super-resolution by leveraging 2D image super-resolution techniques [S1, unofficial]. Unlike standard NeRF implementations that rely on dense 3D voxel grids or large MLPs for all features, this method decouples spatial geometry and view-dependent appearance into explicit 2D planes, allowing the use of established 2D SR architectures for upscaling [S1, unofficial]. It sets itself apart from siblings like [HexPlane](11358-hexplane.md) by using exactly four axis-aligned planes (three positional, one directional) rather than six, and from [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md) by maintaining a dense, fixed-resolution plane structure optimized for low-resolution input scaling.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point $\mathbf{p} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d}$ (defined by azimuth $\theta$ and elevation $\phi$).
*   **Output:** Volume density $\sigma$ and RGB radiance $\mathbf{c} = \{r, g, b\}$.

**Data Structures:**
The scene representation $P_s$ consists of four planes, each with dimensions $H \times W \times C$:
1.  **Positional Planes:** Three planes $P_{xy}, P_{xz}, P_{yz}$ aligned with the coordinate axes. Each has resolution $N \times N \times C$.
2.  **View-Direction Plane:** One plane $P_{dir}$ encoding view-dependent appearance. It has resolution $N_{dir} \times N_{dir} \times C$.

Typical parameter values for low-resolution (LR) training (e.g., 100px images) are $N=200$, $N_{dir}=32$, and channel depth $C=48$ [S1, unofficial]. The view-direction plane resolution is strictly smaller than positional planes ($N_{dir} < N$) because each pixel's information is relevant to only one point on $P_{dir}$ (constant viewing direction along a ray), whereas it is relevant to many points on positional planes [S1, unofficial].

**Decoders:**
Two fully-connected MLPs are used, both with 128 channels and 4 layers [S1, unofficial]:
1.  **Coarse Decoder ($D_c$):** Predicts density $\sigma$.
2.  **Fine Decoder ($D_f$):** Predicts density $\sigma$ (used for super-resolution refinement).
3.  **Radiance Decoder ($D_r$):** Predicts RGB radiance. Identical architecture to density decoders (128 channels, 4 layers) [S1, unofficial].

**Algorithm:**
1.  **Positional Feature Retrieval:** For point $\mathbf{p}=(x,y,z)$, project $\mathbf{p}$ onto the three positional planes to get 2D coordinates. Retrieve feature vectors $v_{xy}, v_{xz}, v_{yz}$ via bilinear interpolation from $P_{xy}, P_{xz}, P_{yz}$.
2.  **Directional Feature Retrieval:** Compute viewing direction angles $(\theta_p, \phi_p)$ for $\mathbf{p}$. Project $(\theta_p, \phi_p)$ onto $P_{dir}$. Retrieve feature vector $v_{pdir}$ via bilinear interpolation [S1, unofficial].
3.  **Feature Aggregation:** Compute the positional feature vector $v_p$ as the average of the positional plane features:
    $$
    v_p = \frac{1}{3} (v_{xy} + v_{xz} + v_{yz})
    $$
4.  **Density Prediction:** Feed $v_p$ into the coarse decoder $D_c$ (and fine decoder $D_f$ during SR training) to obtain density $\sigma$:
    $$
    \sigma = D_c(v_p)
    $$
    Note: $D_c$ and $D_f$ are shared across scenes [S1, unofficial].
5.  **Radiance Prediction:** Concatenate all four feature vectors to form the input for the radiance decoder:
    $$
    v_{concat} = [v_{xy}; v_{xz}; v_{yz}; v_{pdir}]
    $$
    Compute radiance $\mathbf{c}$:
    $$
    \mathbf{c} = D_r(v_{concat})
    $$
    The view-direction plane $P_{dir}$ affects only radiance, not density, as volume density is view-independent in the real world [S1, unofficial].
6.  **Rendering:** Use the predicted $\sigma$ and $\mathbf{c}$ in the standard volumetric rendering integral [Parent].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def quadri_plane_query(p, d, planes, N, N_dir, C):
    """
    Query the quadri-plane representation for a single point.
    
    Args:
        p: tuple (x, y, z), 3D point coordinates.
        d: tuple (theta, phi), viewing direction in spherical coordinates.
        planes: dict containing 'P_xy', 'P_xz', 'P_yz' (N x N x C) and 'P_dir' (N_dir x N_dir x C).
        N: resolution of positional planes.
        N_dir: resolution of view-direction plane.
        C: channel depth.
        
    Returns:
        sigma: float, volume density.
        c: tuple (r, g, b), radiance.
    """
    x, y, z = p
    theta, phi = d
    
    # 1. Retrieve positional features via bilinear interpolation
    # Coordinates are normalized to [0, 1] or mapped to plane indices. 
    # Assuming planes are indexed [0..N-1] and [0..N_dir-1].
    
    # Helper for bilinear interpolation on a 2D plane with channels
    def bilinear_interp(plane_2d, u, v):
        # u, v are normalized coordinates [0, 1]
        # Implementation detail: map u,v to pixel indices and interpolate
        # This is a placeholder for the specific interpolation logic
        # which depends on how the plane coordinates are mapped.
        # Standard NeRF-style mapping: coord -> [0,1] -> pixel center
        pass 

    # Map 3D coords to 2D plane coords. 
    # Standard mapping: x,y,z in [-1, 1] or [0, 1]. 
    # Assuming input p is in [-1, 1] range for simplicity, mapping to [0, 1] for plane lookup.
    def map_to_plane_coord(val):
        return (val + 1.0) / 2.0

    u_xy = (map_to_plane_coord(x), map_to_plane_coord(y))
    u_xz = (map_to_plane_coord(x), map_to_plane_coord(z))
    u_yz = (map_to_plane_coord(y), map_to_plane_coord(z))
    
    v_xy = bilinear_interp(planes['P_xy'], u_xy[0], u_xy[1])
    v_xz = bilinear_interp(planes['P_xz'], u_xz[0], u_xz[1])
    v_yz = bilinear_interp(planes['P_yz'], u_yz[0], u_yz[1])
    
    # 2. Retrieve view-direction feature
    # Map theta, phi to [0, 1] range for P_dir lookup
    # Theta: [0, 2pi], Phi: [0, pi]
    u_dir = (theta / (2 * math.pi), phi / math.pi)
    v_pdir = bilinear_interp(planes['P_dir'], u_dir[0], u_dir[1])
    
    # 3. Aggregate positional features
    # Average of the three positional vectors
    v_p = [0.0] * C
    for k in range(C):
        v_p[k] = (v_xy[k] + v_xz[k] + v_yz[k]) / 3.0
        
    # 4. Predict Density
    # Input to D_c is v_p
    sigma = density_decoder(v_p)
    
    # 5. Predict Radiance
    # Concatenate all 4 vectors
    v_concat = []
    for k in range(C):
        v_concat.extend([v_xy[k], v_xz[k], v_yz[k], v_pdir[k]])
        
    c = radiance_decoder(v_concat)
    
    return sigma, c

def density_decoder(features):
    """
    4-layer MLP with 128 channels.
    Input: features (C,)
    Output: scalar sigma
    """
    # Layer 1
    h = relu(linear(features, 128))
    # Layer 2
    h = relu(linear(h, 128))
    # Layer 3
    h = relu(linear(h, 128))
    # Layer 4 (Output)
    sigma = linear(h, 1)
    return sigma

def radiance_decoder(features):
    """
    4-layer MLP with 128 channels.
    Input: features (4*C,)
    Output: vector (3,) for RGB
    """
    # Layer 1
    h = relu(linear(features, 128))
    # Layer 2
    h = relu(linear(h, 128))
    # Layer 3
    h = relu(linear(h, 128))
    # Layer 4 (Output)
    rgb = linear(h, 3)
    return rgb
```

**Edge Cases and Pitfalls:**
*   **Plane Resolution:** If $N$ is too small, high visual frequencies cannot be represented. If $N$ is too large, plane regions may have insufficient information from LR training images, causing artifacts [S1, unofficial].
*   **View-Direction Dependency:** Omitting $P_{dir}$ significantly degrades performance, especially for shiny/reflective objects with non-uniform lighting (e.g., PSNR drop from 29.6 dB to 23.7 dB in specific contexts) [S1, unofficial].

## Operating characteristics
*   **Memory Complexity:** Dominated by the four planes. Total memory is $4 \times N^2 \times C$ (positional) + $N_{dir}^2 \times C$ (directional). For $N=200, N_{dir}=32, C=48$, this is approximately $4 \times 40000 \times 48 + 1024 \times 48 \approx 7.7$ MB, which is significantly smaller than dense 3D voxel grids or large MLP weights for high-resolution scenes.
*   **Time Complexity:** Query time is dominated by bilinear interpolations and small MLP evaluations (4 layers, 128 channels). This is much faster than large MLPs used in standard NeRF.
*   **Strengths:** Enables efficient volumetric super-resolution by allowing 2D SR networks (like EDSR) to upsample the feature planes directly. Computationally efficient for novel view synthesis.
*   **Limitations:** Performance is sensitive to plane resolution $N$. Requires careful tuning of $N$ relative to input image resolution. Less flexible for arbitrary geometric structures compared to voxel-based methods if not properly regularized.
*   **When to use:** Ideal for scenarios requiring super-resolution of 3D scenes from low-resolution inputs, or when computational efficiency for rendering is critical.
*   **When not to use:** For scenes with extremely complex, high-frequency geometry that cannot be captured by 2D planes, or when high-resolution training data is available and super-resolution is not needed.

## Background
Quadri-Plane Radiance Fields were introduced in "Neural Volume Super-Resolution" (2022) [S1, unofficial]. The method builds on the feature plane representation proposed by Chan et al. for human faces [S1, unofficial], extending it to general scenes by adding a view-direction plane $P_{dir}$ and using coarse/fine decoders. It leverages advances in 2D image super-resolution to improve volumetric rendering quality from low-resolution inputs.

## Sources
[S1] Neural Volume Super-Resolution, https://doi.org/10.48550/arXiv.2212.04666, fetched 2026-10-30 (unofficial)
