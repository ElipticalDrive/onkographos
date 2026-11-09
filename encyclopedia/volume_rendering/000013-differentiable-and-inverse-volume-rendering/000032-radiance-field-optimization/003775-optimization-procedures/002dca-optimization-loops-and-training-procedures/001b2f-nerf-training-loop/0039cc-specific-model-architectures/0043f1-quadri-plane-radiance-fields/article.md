# Quadri-Plane Radiance Fields

Also called: 4-Plane Representation, Quadri-Plane Model

## Overview
Quadri-Plane Radiance Fields represent a 3D scene using four multi-channel 2D feature planes coupled with a lightweight decoder MLP, designed to enable computationally efficient volumetric super-resolution by leveraging 2D image super-resolution techniques [S1, unofficial]. It diverges from typical neural rendering schemes by replacing 3D volumetric grids or coordinate-based MLPs with explicit feature vectors arranged in planes, significantly reducing memory usage while maintaining high-fidelity reconstruction [S1, unofficial].

## How it works
The method computes the radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ as defined in [Specific Model Architectures](8585-specific-model-architectures.md). The architectural variation lies in the feature extraction and decoding stages.

### Inputs and Outputs
*   **Inputs:** A 3D query point $\mathbf{p} \in \mathbb{R}^3$ and its viewing direction defined by azimuth $\theta_p$ and elevation $\phi_p$.
*   **Outputs:** Volume density $\sigma$ and RGB radiance $\mathbf{c} = \{r, g, b\}$.

### Data Structures
The scene representation $P_s$ consists of four 2D feature planes:
1.  **Positional Planes:** Three orthogonal planes $P_{xy}, P_{xz}, P_{yz}$.
    *   Dimensions: $N \times N \times C$.
    *   $N$: Spatial resolution (e.g., 200 for low-resolution training).
    *   $C$: Channel dimension (e.g., 48).
2.  **View-Direction Plane:** One plane $P_{dir}$.
    *   Dimensions: $N_{dir} \times N_{dir} \times C$.
    *   $N_{dir}$: Resolution for viewing direction (e.g., 32).
    *   Constraint: $N_{dir} < N$ because viewing direction information is less dense per pixel than spatial information [S1, unofficial].

### Decoders
Two shared, fully-connected MLPs are used:
1.  **Coarse Decoder ($D_c$):** Outputs density $\sigma$.
    *   Architecture: 4 layers, 128 channels per layer [S1, unofficial].
2.  **Fine Decoder ($D_f$):** Outputs density $\sigma$ (used for super-resolution contexts) or combined with $D_c$ for multi-scale rendering.
    *   Architecture: 4 layers, 128 channels per layer [S1, unofficial].
3.  **Radiance Decoder ($D_r$):** Outputs RGB radiance.
    *   Architecture: 4 layers, 128 channels per layer [S1, unofficial].

### Algorithm Steps

1.  **Feature Extraction from Positional Planes:**
    For query point $\mathbf{p} = (x, y, z)$, retrieve feature vectors from the three positional planes via bilinear interpolation:
    $$
    v_{p,xy} = \text{BilinearInterpolate}(P_{xy}, x, y)
    $$
    $$
    v_{p,xz} = \text{BilinearInterpolate}(P_{xz}, x, z)
    $$
    $$
    v_{p,yz} = \text{BilinearInterpolate}(P_{yz}, y, z)
    $$
    Aggregate these into a single positional feature vector $v_p$ by averaging:
    $$
    v_p = \frac{1}{3} (v_{p,xy} + v_{p,xz} + v_{p,yz})
    $$

2.  **Feature Extraction from View-Direction Plane:**
    Determine the viewing direction $(\theta_p, \phi_p)$ for point $\mathbf{p}$ relative to the camera. Retrieve a complementary feature vector from $P_{dir}$:
    $$
    v_{p,dir} = \text{BilinearInterpolate}(P_{dir}, \theta_p, \phi_p)
    $$
    Note: $\theta_p$ and $\phi_p$ must be normalized or mapped to the $[0, 1]$ range of the plane coordinates [S1, unofficial].

3.  **Density Computation:**
    Compute density $\sigma$ using the coarse decoder $D_c$ (and optionally fine decoder $D_f$ in super-resolution variants). The input is the averaged positional feature vector $v_p$:
    $$
    \sigma = D_c(v_p)
    $$
    *Note: The view-direction feature $v_{p,dir}$ is NOT used for density, as volume density is view-independent in the real world [S1, unofficial].*

4.  **Radiance Computation:**
    Compute radiance $\mathbf{c}$ using the radiance decoder $D_r$. The input is the concatenation of the positional feature vector and the view-direction feature vector:
    $$
    v_{concat} = \text{Concat}(v_p, v_{p,dir})
    $$
    $$
    \mathbf{c} = D_r(v_{concat})
    $$

5.  **Rendering:**
    Render the image using the standard volume rendering integral defined in [Specific Model Architectures](8585-specific-model-architectures.md), summing $\sigma_i$ and $\mathbf{c}_i$ along each ray.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def bilinear_interpolate(plane, u, v):
    """
    Performs bilinear interpolation on a 2D plane at normalized coordinates u, v.
    plane: numpy array of shape (H, W, C)
    u, v: float in [0, 1]
    Returns: numpy array of shape (C,)
    """
    H, W, C = plane.shape
    # Map [0,1] to pixel coordinates
    u_px = u * (W - 1)
    v_px = v * (H - 1)
    
    # Floor and ceil indices
    u0, v0 = int(np.floor(u_px)), int(np.floor(v_px))
    u1, v1 = min(u0 + 1, W - 1), min(v0 + 1, H - 1)
    
    # Interpolation weights
    du = u_px - u0
    dv = v_px - v0
    
    # Get corner values
    q00 = plane[v0, u0, :]
    q01 = plane[v0, u1, :]
    q10 = plane[v1, u0, :]
    q11 = plane[v1, u1, :]
    
    # Bilinear interpolation
    top = q00 * (1 - du) + q01 * du
    bottom = q10 * (1 - du) + q11 * du
    result = top * (1 - dv) + bottom * dv
    
    return result

def quadri_plane_query(p, direction, planes, decoders):
    """
    Queries the Quadri-Plane Radiance Field.
    
    Args:
        p: numpy array (3,), 3D point coordinates.
        direction: numpy array (3,), viewing direction vector.
        planes: dict containing 'P_xy', 'P_xz', 'P_yz', 'P_dir'.
                Each is a numpy array (N, N, C) or (N_dir, N_dir, C).
        decoders: dict containing 'D_c' (density), 'D_r' (radiance).
                  Each is a callable MLP taking (C,) and returning scalar or (3,).
                  
    Returns:
        sigma: float, volume density.
        c: numpy array (3,), RGB radiance.
    """
    x, y, z = p
    
    # 1. Extract positional features
    # Assuming planes are normalized to [0,1] or need scaling. 
    # Standard NeRF-like normalization often maps [-1,1] to [0,1] or similar.
    # Here we assume coordinates are already normalized to [0,1] for the plane lookup.
    v_xy = bilinear_interpolate(planes['P_xy'], x, y)
    v_xz = bilinear_interpolate(planes['P_xz'], x, z)
    v_yz = bilinear_interpolate(planes['P_yz'], y, z)
    
    v_p = (v_xy + v_xz + v_yz) / 3.0
    
    # 2. Extract view-direction features
    # Convert direction vector to azimuth (theta) and elevation (phi)
    # theta = atan2(y, x), phi = atan2(z, sqrt(x^2 + y^2))
    # Normalize to [0, 1] for plane lookup.
    # Note: Exact mapping depends on implementation. 
    # Common mapping: theta in [0, 2pi] -> [0,1], phi in [-pi/2, pi/2] -> [0,1]
    theta = np.arctan2(y, x)
    phi = np.arctan2(z, np.sqrt(x**2 + y**2))
    
    # Normalize to [0, 1]
    u_dir = (theta + np.pi) / (2 * np.pi)
    v_dir = (phi + np.pi / 2) / np.pi
    
    v_p_dir = bilinear_interpolate(planes['P_dir'], u_dir, v_dir)
    
    # 3. Compute Density
    sigma = decoders['D_c'](v_p)
    
    # 4. Compute Radiance
    v_concat = np.concatenate([v_p, v_p_dir])
    c = decoders['D_r'](v_concat)
    
    return sigma, c
```

### Parameters and Typical Values
*   **Plane Resolution ($N$):** Determines visual frequency representation. Too small prohibits high frequencies; too large causes artifacts due to insufficient information in LR training images. Typical value for LR: $N=200$ (learned from 100px images) [S1, unofficial].
*   **View-Direction Plane Resolution ($N_{dir}$):** Set smaller than positional planes ($N_{dir} < N$). Typical value: $N_{dir}=32$ [S1, unofficial].
*   **Channels ($C$):** Typical value: $C=48$ [S1, unofficial].
*   **Decoder Layers/Channels:** 4 layers, 128 channels [S1, unofficial].

### Edge Cases and Pitfalls
*   **View-Direction Dependency:** Omitting $P_{dir}$ causes significant performance drops, especially for shiny/reflective objects with non-uniform lighting. Using $P_{dir}$ improves PSNR (e.g., 29.6 dB vs 23.7 dB in specific contexts) [S1, unofficial].
*   **Resolution Trade-off:** Using planes that are too small prohibits the representation of high visual frequencies. Using planes that are too large results in visual artifacts when rendering novel views due to insufficient relevant information in the captured scene images for those plane regions [S1, unofficial].
*   **Coordinate Normalization:** The mapping of 3D coordinates and viewing directions to the 2D plane coordinates must be consistent and normalized to $[0, 1]$ for bilinear interpolation to work correctly.

## Operating characteristics
*   **Memory Complexity:** $O(N^2 C)$ for positional planes and $O(N_{dir}^2 C)$ for the view-direction plane. This is significantly more memory-efficient than 3D voxel grids ($O(V^3)$) for high-resolution scenes.
*   **Time Complexity:** Query time is dominated by bilinear interpolation and small MLP evaluations, making it fast. However, rendering requires sampling many points along rays.
*   **Strengths:** Enables efficient volumetric super-resolution by leveraging 2D SR techniques. Low memory footprint.
*   **Limitations:** Performance is sensitive to plane resolution $N$. Requires careful tuning of $N$ relative to training image resolution. View-direction plane is critical for specular/reflective surfaces.

## Background
Quadri-Plane Radiance Fields were introduced in "Neural Volume Super-Resolution" [S1, unofficial]. The method builds on the feature plane representation proposed by Chan et al. [9] for human faces, extending it to general scenes by adding a view-direction plane and using coarse/fine decoders. It aims to bridge the gap between neural rendering and 2D image super-resolution.

## Sources
[S1] Neural Volume Super-Resolution, https://doi.org/10.48550/arXiv.2212.04666, fetched 2026-10-30 (unofficial)
