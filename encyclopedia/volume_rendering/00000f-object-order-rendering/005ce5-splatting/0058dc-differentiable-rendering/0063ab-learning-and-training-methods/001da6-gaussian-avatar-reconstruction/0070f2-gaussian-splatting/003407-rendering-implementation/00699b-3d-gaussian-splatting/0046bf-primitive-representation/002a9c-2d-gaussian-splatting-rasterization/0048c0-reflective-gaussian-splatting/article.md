# Reflective Gaussian Splatting

Also called: Ref-Gaussian

## Overview
Reflective Gaussian Splatting (Ref-Gaussian) is a framework for the real-time, high-quality rendering of reflective objects with inter-reflection effects [S1, unofficial]. It addresses the limitations of standard 3D Gaussian Splatting in modeling complex surfaces and multi-view inconsistencies [S1, unofficial]. It distinguishes itself from sibling methods like [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md) by utilizing 2D Gaussian primitives for view-consistent geometry [S1, unofficial]. It employs physically based deferred rendering with split-sum approximation to stabilize optimization and enable efficient inter-reflection modeling without heavy Monte Carlo sampling [S1, unofficial].

## How it works
Ref-Gaussian extends the Gaussian splatting framework by replacing 3D ellipsoids with 2D oriented planar Gaussian disks and applying deferred shading on pixel-level feature maps [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Scene representation: A set of $N$ 2D Gaussian primitives, each defined by position $\mathbf{p}$, tangential vectors $\mathbf{t}_u, \mathbf{t}_v$, scaling factors $s_u, s_v$, and material properties [S1, unofficial].
- Environment map or light sources for illumination [G].

**Outputs:**
- Final pixel radiance $L(\omega_o)$ [S1, unofficial].

### Data Structures
Each 2D Gaussian primitive $i$ stores:
- Position: $\mathbf{p}_i \in \mathbb{R}^3$ [S1, unofficial].
- Tangential vectors: $\mathbf{t}_{u,i}, \mathbf{t}_{v,i} \in \mathbb{R}^3$ [S1, unofficial].
- Scaling factors: $s_{u,i}, s_{v,i} \in \mathbb{R}$ [S1, unofficial].
- Material properties: Albedo $\lambda_i \in [0, 1]^3$, metallic $m_i \in [0, 1]$, and roughness $r_i \in [0, 1]$ [S1, unofficial].
- Derived normal: $\mathbf{n}_i = \mathbf{t}_{u,i} \times \mathbf{t}_{v,i}$ [S1, unofficial].

Pixel-level feature maps $\mathbf{X}$ are accumulated during rasterization:
- $\mathbf{X} = [\mathbf{\Lambda}, M, R, N]^T$, where $\mathbf{\Lambda}$ is the albedo map, $M$ is the metallic map, $R$ is the roughness map, and $N$ is the normal map [S1, unofficial].

### Steps

1. **Primitive Definition**:
   Represent the scene using 2D Gaussian primitives to capture detailed geometry and avoid multi-view inconsistencies inherent in 3D Gaussians [S1, unofficial]. The influence of a 2D Gaussian in its local tangent space is defined as [S1, unofficial]:
   $$
   \mathcal{G}(u, v) = \exp\left(-\frac{u^2 + v^2}{2}\right)
   $$
   where $(u, v)$ are coordinates in the local tangent space [S1, unofficial].

2. **Ray-Splat Intersection**:
   Compute the intersection of a ray with a Gaussian splat in homogeneous coordinates [S1, unofficial]. The intersection point $\mathbf{x}$ is calculated as [S1, unofficial]:
   $$
   \mathbf{x} = (xz, yz, z, 1)^T = WH(u, v, 1, 1)^T
   $$
   where $H$ is the $4 \times 4$ transformation matrix for the Gaussian [S1, unofficial]:
   $$
   H = \begin{bmatrix} s_u \mathbf{t}_u & s_v \mathbf{t}_v & \mathbf{0} & \mathbf{p} \end{bmatrix} \in \mathbb{R}^{4 \times 4}
   $$
   Here, $\mathbf{p}$ is the position, $\mathbf{t}_u, \mathbf{t}_v$ are the tangential vectors, and $s_u, s_v$ are the scaling factors [S1, unofficial].

3. **Alpha Blending for Feature Maps**:
   Perform alpha-blending of Gaussian features to create pixel-level feature maps $\mathbf{X}$, treating alpha-blending as a smoothing filter to stabilize optimization [S1, unofficial]. For each pixel, accumulate the features $\mathbf{x}_i$ weighted by opacity $\alpha_i$ [S1, unofficial]:
   $$
   \mathbf{X} = \sum_{i=1}^N \mathbf{x}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
   $$
   where $\mathbf{x}_i = [\lambda_i, m_i, r_i, \mathbf{n}_i]^T$ and $\mathbf{X} = [\mathbf{\Lambda}, M, R, N]^T$ [S1, unofficial]. The opacity $\alpha_i$ is calculated by multiplying the Gaussian's intrinsic opacity $o_i$ with the Gaussian weight determined by the 2D covariance matrix $\Sigma'_i$ [S1, unofficial].

4. **Deferred Shading with Split-Sum Approximation**:
   Apply the rendering equation on the aggregated feature maps $\mathbf{X}$ rather than shading individual Gaussians before blending [S1, unofficial]. The outgoing radiance $L(\omega_o)$ in direction $\omega_o$ is given by [S1, unofficial]:
   $$
   L(\omega_o) = \int_{\Omega} L_i(\omega_i) f(\omega_i, \omega_o) (\omega_i \cdot \mathbf{N}) \, d\omega_i
   $$
   where $f(\omega_i, \omega_o)$ is the Bidirectional Reflectance Distribution Function (BRDF), and $\mathbf{N}$ is the normal from the feature map [S1, unofficial]. To model inter-reflection effects efficiently, use the split-sum approximation of the rendering equation, avoiding heavy Monte Carlo sampling [S1, unofficial].

5. **Visibility Computation**:
   Compute visibility for the specular component by performing ray tracing on an extracted mesh [S1, unofficial]. This approach provides accurate visibility for specular terms, unlike methods that trace rays across Gaussians which can be noisy and slow [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def reflective_gaussian_splatting(gaussians, camera, environment_map):
    H, W = camera.resolution
    Lambda = [[(0.0, 0.0, 0.0) for _ in range(W)] for _ in range(H)]
    M = [[0.0 for _ in range(W)] for _ in range(H)]
    R = [[0.0 for _ in range(W)] for _ in range(H)]
    N = [[(0.0, 0.0, 0.0) for _ in range(W)] for _ in range(H)]
    Alpha = [[0.0 for _ in range(W)] for _ in range(H)]
    CumProd = [[1.0 for _ in range(W)] for _ in range(H)]

    for i, g in enumerate(gaussians):
        Sigma_prime = compute_2d_covariance(g.p, g.tu, g.tv, g.su, g.sv, camera)
        weight = gaussian_weight(Sigma_prime)
        alpha_i = g.opacity * weight
        n = cross(g.tu, g.tv)
        norm_n = math.sqrt(n[0]**2 + n[1]**2 + n[2]**2)
        n = (n[0]/norm_n, n[1]/norm_n, n[2]/norm_n)
        x_i = (g.lambda_, g.m, g.r, n)

        bbox = get_bounding_box(Sigma_prime, camera)
        for y in range(max(0, bbox.y_min), min(H, bbox.y_max)):
            for x in range(max(0, bbox.x_min), min(W, bbox.x_max)):
                dist_sq = compute_distance_sq(x, y, Sigma_prime, camera)
                contrib_weight = math.exp(-0.5 * dist_sq)
                contrib_alpha = alpha_i * contrib_weight
                w = contrib_alpha * CumProd[y][x]
                Lambda[y][x] = (Lambda[y][x][0] + x_i[0] * w,
                                Lambda[y][x][1] + x_i[1] * w,
                                Lambda[y][x][2] + x_i[2] * w)
                M[y][x] += x_i[1] * w
                R[y][x] += x_i[2] * w
                N[y][x] = (N[y][x][0] + x_i[3][0] * w,
                           N[y][x][1] + x_i[3][1] * w,
                           N[y][x][2] + x_i[3][2] * w)
                Alpha[y][x] += w
                CumProd[y][x] *= (1 - w)

    safe_alpha = [[max(Alpha[y][x], 1e-6) for x in range(W)] for y in range(H)]
    for y in range(H):
        for x in range(W):
            Lambda[y][x] = (Lambda[y][x][0]/safe_alpha[y][x],
                            Lambda[y][x][1]/safe_alpha[y][x],
                            Lambda[y][x][2]/safe_alpha[y][x])
            M[y][x] /= safe_alpha[y][x]
            R[y][x] /= safe_alpha[y][x]
            N[y][x] = (N[y][x][0]/safe_alpha[y][x],
                       N[y][x][1]/safe_alpha[y][x],
                       N[y][x][2]/safe_alpha[y][x])

    image = [[(0.0, 0.0, 0.0) for _ in range(W)] for _ in range(H)]
    for y in range(H):
        for x in range(W):
            image[y][x] = compute_brdf_radiance(Lambda[y][x], M[y][x], R[y][x], N[y][x], environment_map)
    return image

def compute_2d_covariance(p, tu, tv, su, sv, camera):
    # Projects 2D Gaussian parameters to screen-space covariance [G]
    pass

def gaussian_weight(Sigma_prime):
    # Returns normalization weight for the 2D covariance [G]
    return 1.0

def compute_distance_sq(x, y, Sigma_prime, camera):
    # Computes squared Mahalanobis distance in screen space [G]
    pass

def get_bounding_box(Sigma_prime, camera):
    # Returns 3-sigma bounding box in screen coordinates [G]
    pass

def compute_brdf_radiance(albedo, metallic, roughness, normal, environment_map):
    # Applies split-sum approximation to the rendering equation [S1, unofficial]
    pass

def cross(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])
```

### Parameters
- **Albedo ($\lambda$)**: Material color property, range $[0, 1]^3$ [S1, unofficial].
- **Metallic ($m$)**: Material metallic property, range $[0, 1]$ [S1, unofficial].
- **Roughness ($r$)**: Material roughness property, range $[0, 1]$ [S1, unofficial].

### Edge Cases and Pitfalls
- **Multi-view inconsistencies**: Standard 3D Gaussians struggle with detailed geometry due to multi-view inconsistencies [S1, unofficial]. Ref-Gaussian uses 2D Gaussians to mitigate this [S1, unofficial].
- **Optimization stability**: Shading directly on Gaussians before alpha-blending can lead to noise in geometry and material [S1, unofficial]. Ref-Gaussian uses deferred shading on pixel-level feature maps to stabilize optimization [S1, unofficial].
- **Visibility accuracy**: Using Monte Carlo sampling across Gaussians for visibility can be slow and noisy [S1, unofficial]. Ref-Gaussian uses ray tracing on an extracted mesh for specular visibility, providing more accurate results [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Real-time rendering is achieved by using the split-sum approximation to avoid heavy Monte Carlo sampling and by using efficient rasterization of 2D Gaussians [S1, unofficial].
- **Memory Complexity**: Requires storage for 2D Gaussian primitives and pixel-level feature maps [S1, unofficial]. The extracted mesh for visibility computation also adds memory overhead [S1, unofficial].
- **Performance**: Ref-Gaussian maintains fast rendering speeds while modeling inter-reflection effects [S1, unofficial]. It is faster than methods using Monte Carlo sampling for visibility [S1, unofficial].
- **Strengths**: High-quality rendering of reflective objects, accurate inter-reflection modeling, stable optimization, real-time performance [S1, unofficial].
- **Limitations**: Requires an extracted mesh for visibility computation, which may add complexity to the pipeline [S1, unofficial]. 2D Gaussians may have limitations in representing complex 3D structures compared to 3D Gaussians in some scenarios [S1, unofficial].
- **When to use**: When rendering reflective objects with complex inter-reflection effects is required, and real-time performance is necessary [S1, unofficial].
- **When not to use**: When the scene does not have significant reflective properties or inter-reflection effects, standard 3D Gaussian Splatting may be sufficient and simpler [S1, unofficial].

## Background
Reflective Gaussian Splatting (Ref-Gaussian) was proposed to address the limitations of existing Gaussian splatting methods in rendering reflective objects with inter-reflection effects [S1, unofficial]. It builds on the work of 3D Gaussian Splatting (Kerbl et al., 2023) and 2D Gaussian Splatting (Huang et al., 2024) [S1, unofficial]. The method was introduced in a paper by Ye et al. (2024) [S1, unofficial]. It differs from GShader (Jiang et al., 2024) by using deferred shading and from RelightableGaussian (Gao et al., 2023) by using ray tracing on an extracted mesh for visibility [S1, unofficial].

## Sources
[S1] Reflective Gaussian Splatting, https://doi.org/10.48550/arxiv.2412.19282, fetched 2026-10-26 (unofficial)
