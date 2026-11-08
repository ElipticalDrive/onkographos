# Spherical Panoramic Gaussian Splatting (SPaGS)

Also called: Fast and Accurate 3D Gaussian Splatting for Spherical Panoramas, HTGS extended to spherical camera model, SPaGS (Spherical Panoramic Gaussian Splatting), SPaGS Ray-Plane Intersection, omnidirectional Ray-Splat Intersection

## Overview
SPaGS is a real-time rendering method for 3D Gaussian Splatting (3DGS) that supports arbitrary spherical camera models, specifically equirectangular panoramas, by replacing the standard local affine approximation with a numerically stable ray-splat intersection approach [S1]. It solves the problem of rendering artifacts and numerical instability inherent in standard 3DGS when applied to wide-field-of-view or distorted panoramic views, where first-order affine projections fail to accurately represent the Gaussian distribution [S1]. Unlike ODGS, which uses suboptimal tangent plane approximations, or 3DGUT, which relies on the Unscented Transform, SPaGS achieves higher quality and speed by representing each viewing ray as the intersection of two perpendicular planes in camera space, thereby avoiding matrix inversion of the covariance [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A scene represented by a set of 3D Gaussian splats; an equirectangular image of width $W$ and height $H$; a background color $\mathbf{c}_{background}$ [S1].
*   **Output:** A rendered equirectangular image with per-pixel color $\mathbf{C}$ [S1].

**Data Structures**
Each 3D Gaussian splat $i$ is defined by the following fields [S1]:
*   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Position (center) of the Gaussian.
*   $\mathbf{q}_i \in \mathbb{R}^4$: Rotation represented as a quaternion.
*   $s_{u,i}, s_{v,i}, s_{w,i} \in \mathbb{R}$: Scale factors along the local axes.
*   $o_i \in \mathbb{R}$: Opacity (scalar).
*   $\mathbf{c}_i$: View-dependent color, represented by Spherical Harmonics (SH) coefficients of degree 3.
*   $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, derived from $\boldsymbol{\mu}_i, \mathbf{q}_i, s_{u,i}, s_{v,i}, s_{w,i}$ [S1].

**Transformation Matrices**
*   $M \in \mathbb{R}^{4 \times 4}$: World-to-camera transformation matrix [S1].
*   $T \in \mathbb{R}^{4 \times 4}$: Local-to-world transformation matrix for a splat, defined by its scaling, rotation, and position parameters [S1].
*   $T^{-1}$: Inverse of the local-to-world matrix, used to transform camera-space vectors into the splat's local coordinate system [S1].

**Numbered Steps**
1.  **Ray Direction Computation:** For each pixel $(x, y)$ in the equirectangular image ($0 \le x < W, 0 \le y < H$), compute the ray direction $\mathbf{d}$ in camera space. The direction is derived from the pixel coordinates assuming the image plane is a unit sphere mapped to equirectangular coordinates [S1]:
    $$
    \begin{aligned}
    \theta &= \frac{2\pi x}{W} \\
    \phi &= \frac{\pi y}{H} \\
    d_x &= \cos(\theta) \cos(\phi) \\
    d_y &= \sin(\phi) \\
    d_z &= \sin(\theta) \cos(\phi)
    \end{aligned}
    $$
    The source defines spherical coordinates $\theta, \phi$ from a camera vector $\mathbf{c}$ as $\theta = \text{atan2}(c_z, c_x)$ and $\phi = \text{asin}(c_y / \|\mathbf{c}\|)$ [S1]. The direction $\mathbf{d}$ corresponds to the vector from the camera origin through the pixel [S1].

2.  **Plane Normal Calculation:** Compute the normals $\mathbf{n}_x$ and $\mathbf{n}_y$ of two perpendicular planes $\pi_x$ and $\pi_y$ that contain the ray direction $\mathbf{d}$ and the camera origin. These planes are computed directly in camera space using the image dimensions $W$ and $H$ [S1]. The source cites Eqs. (6)-(10) for this calculation [S1]. Since the planes contain the camera origin $(0,0,0)^\top$, they are represented in homogeneous coordinates as $\pi_{x/y} = (\mathbf{n}_{x/y}, 0)^\top$ [S1].

3.  **Plane Transformation:** Transform the planes $\pi_x$ and $\pi_y$ from camera space into the local coordinate system of each 3D Gaussian splat $i$. This is achieved by applying the inverse transformation $T^{-1}$ (which incorporates $M$) to the plane representations [S1]. Let the transformed planes in local space be $\pi'_{x,i}$ and $\pi'_{y,i}$ [S1].

4.  **Ray-Splat Intersection:** Perform the intersection test in the splat's local coordinate system. The ray is represented by the intersection of the two transformed planes $\pi'_{x,i}$ and $\pi'_{y,i}$ [S1]. This approach avoids the explicit inversion of the covariance matrix $\boldsymbol{\Sigma}_i$, ensuring numerical stability even for degenerate Gaussians [S1]. Compute the squared Mahalanobis distance $\rho_i(\mathbf{x})^2$ at the intersection point $\mathbf{x}$ [S1].

5.  **Alpha Evaluation:** Calculate the alpha value $\alpha_i$ for the splat at the intersection point using the Gaussian's opacity $o_i$ and the computed distance $\rho_i(\mathbf{x})^2$ [S1]:
    $$
    \alpha_i = o_i \exp\left(-\frac{1}{2} \rho_i(\mathbf{x})^2\right)
    $$
    (Note: The exact form of Eq. (3) is referenced but not fully expanded in the snippet, but this is the standard Gaussian alpha evaluation consistent with the context [S1]).

6.  **Hybrid Transparency Blending:**
    *   **Core:** Sort the contributing splats by depth. Apply standard $\alpha$-blending to the first $K$ splats (the "core") [S1].
    *   **Tail:** Combine the remaining splats (the "tail") without sorting. Compute the combined tail color $\mathbf{c}_{tail}$ and remaining opacity $T_{tail}$ [S1].
    *   **Final Color:** Compute the final pixel color $\mathbf{C}$ [S1]:
        $$
        \mathbf{C} = \mathbf{c}_{tail} + (1 - T_{tail}) \mathbf{c}_{background}
        $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cross(a, b):
    return [a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0]]

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def normalize(v):
    l = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return [v[0]/l, v[1]/l, v[2]/l]

def render_spherical_panorama(splats, W, H, K, c_background):
    """
    Renders an equirectangular panorama using SPaGS.
    
    Args:
        splats: List of Gaussian splats, each with attributes:
                mu (3,), q (4,), scale (3,), opacity (float), sh_coeffs (degree 3),
                T_inv (4x4 matrix as list of lists)
        W: Image width
        H: Image height
        K: Number of splats in the core blend
        c_background: Background color vector [r, g, b]
        
    Returns:
        Image buffer of shape (H, W, 3)
    """
    image = [[0.0, 0.0, 0.0] for _ in range(H)]
    
    for y in range(H):
        for x in range(W):
            # Step 1: Compute ray direction in camera space
            theta = 2.0 * math.pi * x / W
            phi = math.pi * y / H
            d = [math.cos(theta) * math.cos(phi), math.sin(phi), math.sin(theta) * math.cos(phi)]
            
            # Step 2: Compute plane normals nx, ny in camera space
            up = [0.0, 1.0, 0.0]
            if abs(d[1]) > 0.9:
                up = [1.0, 0.0, 0.0]
            nx = normalize(cross(d, up))
            ny = normalize(cross(d, nx))
            
            # Collect splats that might contribute to this pixel (bounding box culling)
            candidates = [s for s in splats if in_tile(s, x, y)]
            
            contributions = []
            
            for s in candidates:
                # Step 3: Transform planes to splat local space
                # P' = (T_inv)^T * P. We only need the upper 3x3 rotation part for normals.
                R_inv = [row[:3] for row in s.T_inv]
                n_prime_x = [dot(R_inv[0], nx), dot(R_inv[1], nx), dot(R_inv[2], nx)]
                n_prime_y = [dot(R_inv[0], ny), dot(R_inv[1], ny), dot(R_inv[2], ny)]
                
                # Ray in local space: origin o_local = T_inv[:3, 3], direction d_local = R_inv @ d
                o_local = [s.T_inv[0][3], s.T_inv[1][3], s.T_inv[2][3]]
                d_local = [dot(R_inv[0], d[0]), dot(R_inv[1], d[1]), dot(R_inv[2], d[2])]
                d_local = normalize(d_local)
                
                # Step 4 & 5: Intersection in local space & Alpha Evaluation
                # Compute alpha at the point on the ray closest to the Gaussian center [G]
                diff = [s.mu[0] - o_local[0], s.mu[1] - o_local[1], s.mu[2] - o_local[2]]
                proj = dot(diff, d_local)
                closest = [o_local[0] + proj * d_local[0], o_local[1] + proj * d_local[1], o_local[2] + proj * d_local[2]]
                vec = [s.mu[0] - closest[0], s.mu[1] - closest[1], s.mu[2] - closest[2]]
                rho_sq = (vec[0]*s.scale[0])**2 + (vec[1]*s.scale[1])**2 + (vec[2]*s.scale[2])**2
                alpha = s.opacity * math.exp(-0.5 * rho_sq)
                
                if alpha > 0.0:
                    # SH color evaluation is standard and not detailed in the source [G]
                    color = evaluate_sh(s.sh_coeffs, d)
                    depth = dot(d_local, s.mu)
                    contributions.append((alpha, color, depth))
            
            # Step 6: Hybrid Blending
            contributions.sort(key=lambda k: k[2])
            
            # Core blending
            core_color = [0.0, 0.0, 0.0]
            core_alpha = 0.0
            for i in range(min(K, len(contributions))):
                a, c, _ = contributions[i]
                core_color = [core_color[j]*(1-a) + c[j]*a for j in range(3)]
                core_alpha += a * (1 - core_alpha)
            
            # Tail blending (unsorted accumulation as per source)
            tail_color = [0.0, 0.0, 0.0]
            tail_alpha = 0.0
            for i in range(K, len(contributions)):
                a, c, _ = contributions[i]
                tail_color = [tail_color[j]*(1-a) + c[j]*a for j in range(3)]
                tail_alpha += a * (1 - tail_alpha)
                
            # Final color: C = ctail + (1 - Ttail) * c_background
            final_color = [tail_color[j] + (1 - tail_alpha) * c_background[j] for j in range(3)]
            final_color = [final_color[j]*(1-core_alpha) + core_color[j]*core_alpha for j in range(3)]
            
            image[y][x] = final_color

    return image

def evaluate_sh(sh_coeffs, direction):
    # Standard SH evaluation [G]
    return [0.0, 0.0, 0.0] # Placeholder for SH math

def in_tile(splat, x, y):
    # Placeholder for bounding box culling [G]
    return True
```

**Parameters**
*   $K$: The number of depth-sorted splats included in the 'core' part of hybrid transparency blending. The remaining splats form the 'tail'. [S1]
*   Degree of Spherical Harmonics: Set to 3 for view-dependent color representation. [S1]

**Edge Cases and Pitfalls**
*   **Degenerate Gaussians:** Standard covariance-based rasterization can become numerically unstable for degenerate Gaussians (e.g., very thin or flat). SPaGS handles this by using the ray-splat intersection via perpendicular planes, which avoids matrix inversion of the covariance. [S1]
*   **Wide-Field-of-View Artifacts:** Standard 3DGS uses a first-order affine approximation for projection, leading to residual errors and noticeable rendering artifacts in wide-FoV views. SPaGS avoids this by using exact ray-splat intersection for spherical cameras. [S1]
*   **Spherical Distortion:** The equirectangular projection causes non-linear distortion. The tangent plane-based bounding box calculation (see Subtopics) is required to efficiently cull splats despite this distortion. [S1]

## Subtopics

### [Tangent Plane-Based Bounding Box Calculation](3425-tangent-plane-based-bounding-box-calculation.md)
A two-stage method to calculate an axis-aligned screen-space bounding box for 3D Gaussians in spherical panoramas, addressing deformation caused by omnidirectional pixel distribution and non-linear spherical projection. It sets itself apart from standard affine approximations by projecting a tangent-space bounding box onto the equirectangular plane, ensuring validity despite spherical distortion.

## Operating characteristics
*   **Time Complexity:** Real-time rendering is achieved. The method replaces the affine approximation with a ray-splat intersection, which is more expensive per splat but allows for efficient culling via the tangent plane-based bounding box [S1].
*   **Memory Complexity:** Similar to standard 3DGS, storing position, rotation, scale, opacity, and SH coefficients [S1].
*   **Measured Performance:** SPaGS achieves sharper reconstructions on large-scale scenes (e.g., Zip-NeRF) compared to other real-time methods. It is faster than 3DGUT and ODGS while providing better quality [S1].
*   **Strengths:** Numerical stability for degenerate Gaussians; accurate rendering for wide-FoV and distorted panoramic inputs; real-time performance [S1].
*   **Limitations:** More complex than standard 3DGS due to the ray-splat intersection and plane transformations [S1].
*   **When to use:** Real-time rendering of 3D Gaussian splats on spherical panoramas, specifically equirectangular images, supporting distorted panoramic inputs [S1].
*   **When not to use:** Standard perspective cameras without distortion, where standard 3DGS is sufficient and faster [S1].

## Background
SPaGS was introduced by Huang et al. (2024) and Bai et al. (2024) in the context of error analysis and optimal projection strategies for 3DGS [S1]. It extends the approach of Hahlbohm et al. [HFW*25] to spherical camera models. The method addresses the limitations of standard 3DGS, which only supports undistorted perspective camera models, by providing a numerically stable ray-splat intersection approach [S1]. It is used for real-time rendering of 3D Gaussian splats on spherical panoramas [S1].

## Sources
[S1] SPaGS: Fast and Accurate 3D Gaussian Splatting for Spherical Panoramas, https://doi.org/10.1111/cgf.70171, fetched 2026-11-01
