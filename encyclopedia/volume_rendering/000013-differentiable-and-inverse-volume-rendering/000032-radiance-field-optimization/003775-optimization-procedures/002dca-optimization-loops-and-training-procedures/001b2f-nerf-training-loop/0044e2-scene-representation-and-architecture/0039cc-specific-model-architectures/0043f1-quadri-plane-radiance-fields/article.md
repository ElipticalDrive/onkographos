# Quadri-Plane Radiance Fields

Also called: 4-Plane Representation, Quadri-Plane Model

## Overview
Quadri-Plane Radiance Fields represent a 3D scene using four multi-channel 2D feature planes coupled with a lightweight decoder MLP, designed to enable computationally efficient volumetric super-resolution by leveraging 2D image super-resolution techniques [S1, unofficial]. Unlike standard NeRF implementations that use a single dense 3D volume or hash grids, this method decomposes the scene into explicit 2D planes, allowing for resolution scaling independent of the 3D sampling density [S1, unofficial]. It diverges from sibling architectures like [AdaNeRF](6195-adanerf.md) and [SinNeRF](6199-sinnerf.md) by explicitly separating positional and viewing-direction features into distinct planar structures, which significantly improves reconstruction quality for shiny or reflective objects under non-uniform lighting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D position $\mathbf{p} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^3$ (defined by azimuth $\theta$ and elevation $\phi$).
*   **Output:** Volume density $\sigma \in \mathbb{R}$ and RGB radiance $\mathbf{c} \in \mathbb{R}^3$.

**Data Structures:**
The scene representation $P_s$ consists of four planes, each with dimensions $H \times W \times C$:
1.  **Positional Planes:** Three orthogonal planes $P_{xy}, P_{xz}, P_{yz}$ with resolution $N \times N \times C$.
2.  **View-Direction Plane:** One plane $P_{dir}$ with resolution $N_{dir} \times N_{dir} \times C$.
*   **Parameters:** Typical values for low-resolution training are $N=200$, $N_{dir}=32$, and channel depth $C=48$ [S1, unofficial]. The view-direction plane resolution is strictly smaller than positional planes ($N_{dir} < N$) because a single viewing direction applies to all points along a ray, whereas a single spatial point is viewed from many directions [S1, unofficial].

**Decoders:**
Two fully-connected MLPs are used:
1.  **Coarse Decoder ($D_c$):** 4 layers, 128 channels. Inputs: Average of positional feature vectors. Outputs: Coarse density $\sigma_c$.
2.  **Fine Decoder ($D_f$):** 4 layers, 128 channels. Inputs: Average of positional feature vectors. Outputs: Fine density $\sigma_f$ and radiance $\mathbf{c}$.
*   Note: The radiance decoder concatenates all four feature vectors (3 positional + 1 view-direction), while the density decoders use only the average of the three positional vectors, as density is view-independent [S1, unofficial].

**Algorithm:**
1.  **Feature Retrieval:** For a query point $\mathbf{p}$, retrieve feature vectors from the three positional planes via bilinear interpolation:
    $$
    v_{p,xy} = \text{Bilinear}(P_{xy}, p_{xy}), \quad v_{p,xz} = \text{Bilinear}(P_{xz}, p_{xz}), \quad v_{p,yz} = \text{Bilinear}(P_{yz}, p_{yz})
    $$
    Compute the aggregated positional feature vector:
    $$
    v_p = \frac{1}{3} (v_{p,xy} + v_{p,xz} + v_{p,yz})
    $$
2.  **View-Direction Feature Retrieval:** Convert viewing direction $\mathbf{d}$ to spherical coordinates $(\theta_p, \phi_p)$. Retrieve the view-direction feature vector:
    $$
    v_{p,dir} = \text{Bilinear}(P_{dir}, (\theta_p, \phi_p))
    $$
3.  **Density Prediction:** Pass $v_p$ to the coarse and fine decoders:
    $$
    \sigma_c = D_c(v_p), \quad \sigma_f = D_f(v_p)
    $$
4.  **Radiance Prediction:** Concatenate all feature vectors and pass to the fine decoder (or a dedicated radiance MLP with identical architecture):
    $$
    \mathbf{c} = D_f([v_p, v_{p,dir}])
    $$
    *(Note: The source specifies $D_f$ computes radiance from the concatenated input, implying $D_f$ has two modes or the radiance output is derived from the same network structure but different input concatenation. We assume the fine decoder handles both or a parallel MLP with identical specs is used for radiance.)*
5.  **Rendering:** Use $\sigma_c, \sigma_f, \mathbf{c}$ in the standard volumetric rendering equation [Parent]:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def quadri_plane_query(p, d, planes, N, N_dir, C):
    """
    Query the quadri-plane representation for density and color.
    
    Args:
        p: tuple (x, y, z), 3D position.
        d: tuple (theta, phi), viewing direction in spherical coords.
        planes: dict containing 'P_xy', 'P_xz', 'P_yz', 'P_dir'.
                Each is a 3D tensor (H, W, C).
        N: resolution of positional planes.
        N_dir: resolution of view-direction plane.
        C: channel dimension.
        
    Returns:
        sigma: float, volume density.
        color: tuple (r, g, b), radiance.
    """
    
    # 1. Retrieve positional features via bilinear interpolation
    # Normalize coordinates to [0, 1] for interpolation if planes are indexed 0..N-1
    # Assuming bilinear function handles normalization or expects normalized coords.
    v_xy = bilinear_interpolate(planes['P_xy'], (p[0], p[1]), N, C)
    v_xz = bilinear_interpolate(planes['P_xz'], (p[0], p[2]), N, C)
    v_yz = bilinear_interpolate(planes['P_yz'], (p[1], p[2]), N, C)
    
    # Aggregate positional features
    v_p = (v_xy + v_xz + v_yz) / 3.0
    
    # 2. Retrieve view-direction feature
    # d is (theta, phi). Map to plane coordinates.
    # Typically theta in [0, 2pi], phi in [0, pi].
    # Map to [0, 1] range for interpolation.
    u_dir = d[0] / (2 * math.pi)
    v_dir = d[1] / math.pi
    
    v_dir_feat = bilinear_interpolate(planes['P_dir'], (u_dir, v_dir), N_dir, C)
    
    # 3. Decode Density
    # Coarse and Fine decoders are identical MLPs (4 layers, 128 channels)
    # Input is v_p
    sigma_c = decode_density(v_p)
    sigma_f = decode_density(v_p)
    
    # 4. Decode Radiance
    # Input is concatenation of v_p and v_dir_feat
    v_concat = concatenate(v_p, v_dir_feat)
    color = decode_radiance(v_concat)
    
    return sigma_f, color

def bilinear_interpolate(plane, coords, resolution, channels):
    """
    Perform bilinear interpolation on a 2D plane.
    coords: (u, v) in [0, 1].
    resolution: H (and W for square planes).
    """
    u, v = coords
    # Map to pixel coordinates
    u_px = u * (resolution - 1)
    v_px = v * (resolution - 1)
    
    # Get integer and fractional parts
    u0 = int(math.floor(u_px))
    v0 = int(math.floor(v_px))
    u1 = u0 + 1
    v1 = v0 + 1
    
    # Clamp to boundaries
    u0 = max(0, min(u0, resolution - 1))
    u1 = max(0, min(u1, resolution - 1))
    v0 = max(0, min(v0, resolution - 1))
    v1 = max(0, min(v1, resolution - 1))
    
    # Fractional weights
    w_u = u_px - u0
    w_v = v_px - v0
    
    # Retrieve 4 corners
    # plane is (H, W, C) -> access as plane[v, u]
    c00 = plane[v0, u0] # Shape (C,)
    c01 = plane[v0, u1]
    c10 = plane[v1, u0]
    c11 = plane[v1, u1]
    
    # Interpolate
    # Linear interp in u
    c0 = c00 * (1 - w_u) + c01 * w_u
    c1 = c10 * (1 - w_u) + c11 * w_u
    
    # Linear interp in v
    result = c0 * (1 - w_v) + c1 * w_v
    
    return result

def decode_density(v_p):
    """
    MLP: 4 layers, 128 channels.
    Input: v_p (C,)
    Output: scalar sigma
    """
    # Placeholder for MLP implementation details not fully specified in source
    # Assuming standard ReLU activations and linear layers
    # Layer 1: Linear(C, 128) -> ReLU
    # Layer 2: Linear(128, 128) -> ReLU
    # Layer 3: Linear(128, 128) -> ReLU
    # Layer 4: Linear(128, 1) -> Exp (to ensure positive density)
    # Note: Source says "calculate density value sigma... using fully-connected... MLPs"
    # It does not specify activation functions, but standard NeRF uses Exp for sigma.
    # [G] Assuming standard NeRF density activation (Exp) for positivity.
    h = v_p
    h = relu(linear(h, 128))
    h = relu(linear(h, 128))
    h = relu(linear(h, 128))
    sigma = exp(linear(h, 1))
    return sigma

def decode_radiance(v_concat):
    """
    MLP: 4 layers, 128 channels.
    Input: v_concat (2C,)
    Output: vector color (3,)
    """
    h = v_concat
    h = relu(linear(h, 128))
    h = relu(linear(h, 128))
    h = relu(linear(h, 128))
    color = linear(h, 3) # Sigmoid or raw? Source says "radiance values {r,g,b}".
                         # Usually clipped or sigmoided. [G] Assuming raw output, 
                         # clipped to [0,1] during rendering or loss.
    return color
```

**Edge Cases and Pitfalls:**
*   **Plane Resolution:** If $N$ is too small, high visual frequencies are prohibited. If $N$ is too large, regions of the plane receive insufficient information from low-resolution training images, causing artifacts [S1, unofficial].
*   **View-Direction Dependency:** Omitting $P_{dir}$ causes significant performance drops (e.g., PSNR 23.7 dB vs 29.6 dB) for shiny/reflective objects [S1, unofficial].
*   **Coordinate Mapping:** The view-direction plane uses spherical coordinates $(\theta, \phi)$. Care must be taken to map these correctly to the 2D plane indices [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Query time is dominated by bilinear interpolation on 2D planes and the small MLP decoders. It is significantly faster than dense 3D grid lookups or hash grid traversals for high-resolution scenes because the planes are 2D and small ($N=200$).
*   **Memory Complexity:** Memory usage is $4 \times N \times N \times C$ for positional planes and $N_{dir} \times N_{dir} \times C$ for the view-direction plane. For $N=200, N_{dir}=32, C=48$, this is roughly $4 \times 200^2 \times 48 + 32^2 \times 48 \approx 7.8$ MB, which is very compact compared to dense voxel grids.
*   **Performance:** Achieves high PSNR (e.g., 29.6 dB) on low-resolution inputs when super-resolved. The method is specifically optimized for volumetric super-resolution [S1, unofficial].
*   **Strengths:** Efficient memory footprint; leverages 2D SR techniques; handles view-dependent effects (specularities) via $P_{dir}$.
*   **Limitations:** Performance degrades if plane resolution is mismatched to input image resolution; requires careful tuning of $N$ and $N_{dir}$ [S1, unofficial].

## Background
The Quadri-Plane Radiance Field was introduced in "Neural Volume Super-Resolution" [S1, unofficial]. It builds on the feature plane representation proposed by Chan et al. [9] for human faces, extending it to general scenes by adding a view-direction plane and using coarse/fine decoders [S1, unofficial]. The method was designed to address the computational cost of volumetric super-resolution by exploiting the efficiency of 2D image processing.

## Sources
[S1] Neural Volume Super-Resolution, https://doi.org/10.48550/arXiv.2212.04666, fetched 2026-10-30 (unofficial)
