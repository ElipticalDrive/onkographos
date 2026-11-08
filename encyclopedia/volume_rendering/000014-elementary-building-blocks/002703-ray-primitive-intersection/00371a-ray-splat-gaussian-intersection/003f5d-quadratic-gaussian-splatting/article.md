# Quadratic Gaussian Splatting

Also called: Quadratic Gaussian Model

## Overview
Quadratic Gaussian Splatting (QGS) is a surface reconstruction method that replaces static Gaussian primitives with deformable quadric surfaces (specifically paraboloids) to capture intricate geometry [S1]. Unlike standard Gaussian Splatting which uses Euclidean distance for density modeling, QGS employs geodesic distance-based density distributions to maintain spatial coherence during deformation [S1]. It sets itself apart from its siblings by generalizing 3D Gaussian primitives to higher-order surfaces, allowing adaptive curvature modeling while retaining the efficiency of ray-splat intersection [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A sparse point cloud, posed images, camera intrinsics ($\mathbf{K}$), and relative poses ($\mathbf{R}_{rn}, \mathbf{T}_{rn}$).
*   **Outputs:** A set of quadratic primitives defined by conic matrices, rendered color/depth/normal/curvature maps, and a fused mesh via TSDF.

**Data Structures**
Each primitive is defined by a symmetric $4 \times 4$ conic matrix $\mathbf{Q} \in \mathbb{R}^{4 \times 4}$ and scale parameters $\mathbf{s} = [s_1, s_2, s_3]$.
The matrix $\mathbf{Q}$ is stored as 10 unique coefficients due to symmetry:
$$ \mathbf{Q} = \begin{bmatrix} A & B & C & D \\ B & E & F & G \\ C & F & H & I \\ D & G & I & J \end{bmatrix} $$
The primitive is constrained to be a paraboloid (or degenerate planar ellipse). The relationship between the object space parameters and the conic matrix is:
$$ \mathbf{Q} = \mathbf{T}^{-T} \mathbf{D} \mathbf{T}^{-1} $$
where $\mathbf{T}$ is the transformation matrix (position and orientation) and $\mathbf{D}$ is the diagonal shape matrix. For a paraboloid, $\mathbf{D}$ is derived from signed scales $s_1, s_2, s_3$ as defined in the formulas below.

**Parameters**
*   **Scale:** $s(x, t) = \tanh(t) \exp(x)$. This introduces a sign, allowing smooth transition between elliptic and hyperbolic paraboloids [S1].
*   **Gradient Threshold:** 0.3 [S1].
*   **Percent Dense Value:** 0.001 [S1].
*   **TSDF Voxel Size:** 0.004 [S1].
*   **TSDF Truncation Threshold:** 0.02 [S1].

**Formulas**
1.  **Paraboloid Equation:**
    In local coordinates $(\tilde{x}, \tilde{y}, \tilde{z})$, the implicit surface is:
    $$ f(\tilde{x}, \tilde{y}, \tilde{z}) = \frac{\text{sign}(s_1)}{s_1^2} \tilde{x}^2 + \frac{\text{sign}(s_2)}{s_2^2} \tilde{y}^2 - \frac{1}{s_3} \tilde{z} = 0 $$
    Explicit form:
    $$ \tilde{z}(\tilde{x}, \tilde{y}) = s_3 \left( \frac{\text{sign}(s_1)}{s_1^2} \tilde{x}^2 + \frac{\text{sign}(s_2)}{s_2^2} \tilde{y}^2 \right) $$

2.  **Geodesic Distance and Gaussian Weight:**
    Convert to cylindrical coordinates: $\tilde{x} = \rho \cos \theta, \tilde{y} = \rho \sin \theta$.
    $$ \tilde{z}(\theta, \rho) = a(\theta) \rho^2, \quad \text{where } a(\theta) = s_3 \left( \frac{\text{sign}(s_1) \cos^2 \theta}{s_1^2} + \frac{\text{sign}(s_2) \sin^2 \theta}{s_2^2} \right) $$
    The geodesic distance $l$ from the origin to $\rho_0$ along angle $\theta_0$ is:
    $$ l(a, \rho_0) = \frac{\ln(\sqrt{u^2 + 1} + u) + u\sqrt{u^2 + 1}}{4a}, \quad \text{where } u = 2a\rho_0 $$
    The standard deviation $\sigma(\theta_0)$ in direction $\theta_0$ is:
    $$ \sigma(\theta_0) = \frac{s_1 s_2}{\sqrt{(s_2 \cos \theta_0)^2 + (s_1 \sin \theta_0)^2}} $$
    The Gaussian weight at point $\mathbf{p}_0(\theta_0, \rho_0)$ is:
    $$ G(\mathbf{p}_0) = \exp\left(-\frac{(l(a(\theta_0), \rho_0))^2}{2(\sigma(\theta_0))^2}\right) $$

3.  **Normal and Curvature:**
    Surface normal at $\mathbf{p}_0(\tilde{x}_0, \tilde{y}_0)$:
    $$ \mathbf{n}_0 = \left( \frac{2\text{sign}(s_1)}{s_1^2} \tilde{x}_0, \frac{2\text{sign}(s_2)}{s_2^2} \tilde{y}_0, -\frac{1}{s_3} \right) $$
    Gaussian Curvature at $\mathbf{p}_0$:
    Let $\lambda_1 = \text{sign}(s_1) \cdot s_3/s_1^2$ and $\lambda_2 = \text{sign}(s_2) \cdot s_3/s_2^2$.
    $$ K_0 = \frac{4\lambda_1\lambda_2}{(1 + 4\lambda_1^2\tilde{x}_0^2 + 4\lambda_2^2\tilde{y}_0^2)^2} $$

4.  **Rendering (Alpha Blending):**
    For a ray intersecting $N$ primitives sorted by depth:
    $$ \mathbf{N}, \mathbf{K} = \sum_{i=0}^{N-1} G_i \alpha_i \prod_{j=0}^{i-1} (1 - G_j \alpha_j) \{\mathbf{n}_i, K_i\} $$
    Color is blended similarly using $G_i \alpha_i$ as the opacity term.

5.  **Loss Functions:**
    *   **Depth Distortion:** $\mathcal{L}_d = \sum_{i=0}^{N-1} \sum_{j=0}^{i-1} \omega_i \omega_j (t_i - t_j)^2$, where $\omega_i = \alpha_i T_i$.
    *   **Curvature Guided Normal Consistency:** $\mathcal{L}_{Kn}(u, v) = \lambda_K(K(u, v)) \mathcal{L}_n(u, v)$, where $\mathcal{L}_n = \sum_i \omega_i (1 - \mathbf{n}_i^T \mathbf{N})$ and $\lambda_K(K) = 1 - \text{sigmoid}(\ln(|K| + \epsilon))$.
    *   **Multi-view Regularization:** $\mathcal{L}_{MV} = \frac{1}{V} \sum_{\mathbf{p}_r \in V} \|\mathbf{p}_r - \mathbf{H}_{nr}\mathbf{H}_{rn}\mathbf{p}_r\| + (1 - \text{NCC}(\mathbf{I}_r(\mathbf{p}_r), \mathbf{I}_n(\mathbf{H}_{rn}\mathbf{p}_r)))$.
    *   **Total Loss:** $\mathcal{L} = \mathcal{L}_c + \lambda_d \mathcal{L}_d + \lambda_n \mathcal{L}_{Kn} + \lambda_{Mv} \mathcal{L}_{Mv}$.

**Numbered Steps**
1.  **Initialization:** Initialize primitives from a sparse point cloud. Define $\mathbf{Q}$ using $\mathbf{T}$ and $\mathbf{D}$ derived from scales $s_1, s_2, s_3$.
2.  **Ray-Splat Intersection:**
    *   Transform ray origin $\mathbf{o}$ and direction $\mathbf{d}$ to the primitive's local space: $\hat{\mathbf{o}}, \hat{\mathbf{d}}$.
    *   Define ray in local space: $\hat{\mathbf{p}}(t) = \hat{\mathbf{o}} + t\hat{\mathbf{d}}$.
    *   Substitute $\hat{\mathbf{p}}(t)$ into the paraboloid equation (Eq. 6) to form a quadratic equation in $t$.
    *   Solve for two intersection points $t_n$ and $t_f$ ($t_n \le t_f$).
    *   **Intersection Selection:**
        *   Calculate geodesic distance and $\sigma$ for $\hat{\mathbf{p}}_n$ (corresponding to $t_n$). If distance $\le 3\sigma(\theta_n)$, select $\hat{\mathbf{p}}_n$.
        *   Else, calculate for $\hat{\mathbf{p}}_f$ (corresponding to $t_f$). If distance $\le 3\sigma(\theta_f)$, select $\hat{\mathbf{p}}_f$.
        *   Else, no intersection.
3.  **Per-pixel Resorting:** Sort primitives per pixel to address depth pop-out issues, ensuring smoother normals [S1].
4.  **Rendering:** Compute $G_i$, $\mathbf{n}_i$, $K_i$ for selected intersections. Alpha-blend color, depth, normal, and curvature maps.
5.  **Optimization:** Compute losses $\mathcal{L}_d, \mathcal{L}_{Kn}, \mathcal{L}_{MV}$ and update parameters using gradient descent.
6.  **Mesh Extraction (Post-processing):**
    *   Compute median depth: $t_{\text{median}} = \max t_i \text{ where } T_i > 0.5$.
    *   Fuse depth maps using Truncated Signed Distance Fusion (TSDF) with Open3D [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_paraboloid_params(s1, s2, s3):
    """Compute lambda1, lambda2, and a(theta) coefficients."""
    sign_s1 = 1.0 if s1 >= 0 else -1.0
    sign_s2 = 1.0 if s2 >= 0 else -1.0
    
    lambda1 = sign_s1 * s3 / (s1 ** 2)
    lambda2 = sign_s2 * s3 / (s2 ** 2)
    
    # Coefficients for a(theta) = s3 * (sign(s1)/s1^2 * cos^2 + sign(s2)/s2^2 * sin^2)
    c1 = sign_s1 / (s1 ** 2)
    c2 = sign_s2 / (s2 ** 2)
    
    return lambda1, lambda2, s3 * c1, s3 * c2

def geodesic_distance(a, rho):
    """Compute geodesic distance l(a, rho)."""
    if abs(a) < 1e-9:
        return rho # Degenerate case: Euclidean distance
    u = 2.0 * a * rho
    sqrt_u2_1 = math.sqrt(u**2 + 1.0)
    return (math.log(sqrt_u2_1 + u) + u * sqrt_u2_1) / (4.0 * a)

def gaussian_sigma(s1, s2, theta):
    """Compute standard deviation sigma(theta)."""
    cos_t = math.cos(theta)
    sin_t = math.sin(theta)
    denom_sq = (s2 * cos_t)**2 + (s1 * sin_t)**2
    if denom_sq < 1e-9:
        return 0.0
    return (s1 * s2) / math.sqrt(denom_sq)

def ray_splat_intersection_local(o_local, d_local, s1, s2, s3):
    """
    Intersect ray with paraboloid in local space.
    Returns list of valid intersection points (t, x, y, z, theta, rho).
    """
    # Paraboloid: z = s3 * (c1*x^2 + c2*y^2)
    # Ray: x = ox + dx*t, y = oy + dy*t, z = oz + dz*t
    # Substitute: oz + dz*t = s3 * (c1*(ox+dx*t)^2 + c2*(oy+dy*t)^2)
    
    sign_s1 = 1.0 if s1 >= 0 else -1.0
    sign_s2 = 1.0 if s2 >= 0 else -1.0
    c1 = sign_s1 / (s1 ** 2)
    c2 = sign_s2 / (s2 ** 2)
    
    # Quadratic equation At^2 + Bt + C = 0
    # z = s3*c1*(ox^2 + 2*ox*dx*t + dx^2*t^2) + s3*c2*(oy^2 + 2*oy*dy*t + dy^2*t^2)
    # oz + dz*t = s3*c1*ox^2 + 2*s3*c1*ox*dx*t + s3*c1*dx^2*t^2 + ...
    
    A = s3 * c1 * d_local[0]**2 + s3 * c2 * d_local[1]**2 - d_local[2]
    B = 2 * s3 * c1 * o_local[0] * d_local[0] + 2 * s3 * c2 * o_local[1] * d_local[1] - o_local[2]
    C = s3 * c1 * o_local[0]**2 + s3 * c2 * o_local[1]**2
    
    intersections = []
    
    if abs(A) < 1e-9:
        # Linear equation
        if abs(B) > 1e-9:
            t = -C / B
            x = o_local[0] + d_local[0] * t
            y = o_local[1] + d_local[1] * t
            z = o_local[2] + d_local[2] * t
            intersections.append((t, x, y, z))
        return intersections

    discriminant = B**2 - 4*A*C
    if discriminant < 0:
        return intersections
        
    sqrt_disc = math.sqrt(discriminant)
    t1 = (-B - sqrt_disc) / (2 * A)
    t2 = (-B + sqrt_disc) / (2 * A)
    
    ts = sorted([t1, t2])
    
    for t in ts:
        x = o_local[0] + d_local[0] * t
        y = o_local[1] + d_local[1] * t
        z = o_local[2] + d_local[2] * t
        
        # Calculate rho and theta
        rho = math.sqrt(x**2 + y**2)
        theta = math.atan2(y, x)
        
        # Check if within 3 sigma
        sigma = gaussian_sigma(s1, s2, theta)
        if sigma < 1e-9:
            continue
            
        l = geodesic_distance(
            s3 * (c1 * math.cos(theta)**2 + c2 * math.sin(theta)**2),
            rho
        )
        
        if l <= 3.0 * sigma:
            intersections.append((t, x, y, z, theta, rho))
            
    # Selection logic: prefer nearest valid, then farthest valid
    # The source says: if pn valid, select pn. Else if pf valid, select pf.
    # Since ts is sorted, intersections[0] is pn, intersections[1] is pf (if both valid)
    # But we only want ONE point.
    
    valid_points = []
    for item in intersections:
        t, x, y, z, theta, rho = item
        sigma = gaussian_sigma(s1, s2, theta)
        if sigma < 1e-9: continue
        l = geodesic_distance(
            s3 * (c1 * math.cos(theta)**2 + c2 * math.sin(theta)**2),
            rho
        )
        if l <= 3.0 * sigma:
            valid_points.append((t, x, y, z, theta, rho))
            
    if not valid_points:
        return []
        
    # Select nearest valid
    valid_points.sort(key=lambda k: k[0])
    return [valid_points[0]]

def compute_normal_curvature(x, y, s1, s2, s3):
    sign_s1 = 1.0 if s1 >= 0 else -1.0
    sign_s2 = 1.0 if s2 >= 0 else -1.0
    
    nx = 2.0 * sign_s1 * x / (s1**2)
    ny = 2.0 * sign_s2 * y / (s2**2)
    nz = -1.0 / s3
    
    normal = (nx, ny, nz)
    
    lambda1 = sign_s1 * s3 / (s1**2)
    lambda2 = sign_s2 * s3 / (s2**2)
    
    denom = 1.0 + 4.0 * lambda1**2 * x**2 + 4.0 * lambda2**2 * y**2
    curvature = (4.0 * lambda1 * lambda2) / (denom**2)
    
    return normal, curvature
```

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ per pixel where $N$ is the number of splats, similar to 3DGS, but with higher constant cost due to geodesic distance calculation and quadratic intersection.
*   **Memory Complexity:** Each primitive stores a $4 \times 4$ symmetric matrix (10 floats) plus scales, compared to 3DGS's covariance matrix (6 floats) and scale (3 floats). However, QGS uses fewer primitives for high-curvature surfaces, potentially reducing total memory [S1].
*   **Performance:** On the DTU dataset, QGS reduces geometric error (chamfer distance) by 33% over 2DGS and 27% over GOF [S1]. Experiments were conducted on a single A6000 GPU [S1].
*   **Strengths:** Captures intricate geometry and high-curvature regions better than planar or ellipsoidal primitives. Maintains multi-view consistency and perspective correctness.
*   **Limitations:** More complex intersection and density calculation. Non-convex quadrics require rectangular truncation and approximations for bounding box computation [S1].

## Background
Quadratic Gaussian Splatting (QGS) was proposed by Ziyu Zhang, Binbin Huang, et al. from CASIA, UCAS, The University of Hong Kong, and SenseTime Research [S1]. It was presented at ICCV 2025 [S1]. The method extends 3D Gaussian Splatting [21] and 2D Gaussian Splatting [16] by using second-order geometric primitives (quadrics) to improve surface reconstruction accuracy.

## Sources
[S1] Quadratic Gaussian Splatting: High Quality Surface Reconstruction with Second-Order Geometric Primitives, https://doi.org/10.1109/ICCV51701.2025.02624, fetched 2026-10-30
