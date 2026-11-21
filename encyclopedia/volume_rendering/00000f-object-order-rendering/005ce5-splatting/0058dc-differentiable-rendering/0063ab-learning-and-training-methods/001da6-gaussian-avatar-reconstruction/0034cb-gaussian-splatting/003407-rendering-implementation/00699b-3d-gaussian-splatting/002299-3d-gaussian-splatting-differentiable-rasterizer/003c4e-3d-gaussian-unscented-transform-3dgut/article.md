# 3D Gaussian Unscented Transform (3DGUT)

Also called: 3DGUT Rendering, Sigma point projection, UT-based Gaussian Projection, UT-based projection, Unscented Transform for Gaussian Projection, Unscented Transform projection

## Overview
3D Gaussian Unscented Transform (3DGUT) is a rasterization-based rendering technique for 3D Gaussian particles that replaces the standard Elliptical Weighted Average (EWA) splatting formulation with the Unscented Transform (UT) [S1]. It approximates 3D Gaussian particles using a set of sigma points to enable exact projection under arbitrary nonlinear camera projections, such as those with distortion or rolling shutter effects, while retaining the efficiency of rasterization [S1]. Unlike its sibling [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md), which relies on linearization via Jacobians, 3DGUT avoids Jacobian derivation and approximation errors by approximating the particle itself [S1]. This alignment with ray-tracing formulations also supports secondary rays for phenomena like reflections and refraction [S1].

## How it works
**Inputs:**
- A set of $N$ 3D Gaussian particles. Each particle $i$ is defined by:
  - Mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$ [S1].
  - Covariance matrix $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3\times3}$, decomposed into rotation $\mathbf{R}_i \in SO(3)$ (stored as quaternion $\mathbf{q} \in \mathbb{R}^4$) and scaling $\mathbf{S}_i \in \mathbb{R}^{3\times3}$ (stored as vector $\mathbf{s} \in \mathbb{R}^3$) such that $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$ [S1].
  - Opacity coefficient $\sigma_i \in \mathbb{R}$ [S1].
  - View-dependent radiance represented by spherical harmonics of order $m=3$ [S1].
- Camera parameters: Extrinsic matrix $\mathbf{W} \in SE(3)$ and Intrinsic matrix $\mathbf{D}$ [S1].
- UT hyperparameters: $\alpha, \beta, \kappa$ [S1].

**Outputs:**
- For each particle, the projected 2D mean $\mathbf{v}_{\boldsymbol{\mu}_i}$ and 2D covariance $\boldsymbol{\Sigma}'_i$ [S1].
- A 2D bounding rectangle $\mathbf{r}_i$ for tile-based rasterization culling [S1].

**Data Structures:**
- **Sigma Points:** A set $X = \{\mathbf{x}_i\}_{i=0}^6$ of 7 points in $\mathbb{R}^3$ [S1].
- **Weights:** Two sets of scalar weights, $w_i^\mu$ for the mean and $w_i^\Sigma$ for the covariance, for $i=0..6$ [S1].

**Parameters:**
- $\alpha$: Controls the spread of sigma points around the mean. Default: $1.0$ [S1].
- $\beta$: Incorporates prior knowledge about the distribution. Default: $2.0$ [S1].
- $\kappa$: Scaling parameter. Default: $0.0$ [S1].
- $\lambda$: Derived scaling factor, $\lambda = \alpha^2(3 + \kappa) - 3$ [S1].

**Steps:**

1. **Compute Scaling Factor:**
   Calculate $\lambda = \alpha^2(3 + \kappa) - 3$ [S1].

2. **Generate Sigma Points:**
   Construct 7 sigma points $X = \{\mathbf{x}_i\}_{i=0}^6$ from the 3D Gaussian parameters $\boldsymbol{\mu}$ and $\boldsymbol{\Sigma}$ [S1]. The covariance matrix square root is obtained via the factorization $\boldsymbol{\Sigma} = \mathbf{R}\mathbf{S}\mathbf{S}^T\mathbf{R}^T$ [S1].
   $$
   \mathbf{x}_i = \begin{cases} 
   \boldsymbol{\mu} & \text{for } i = 0 \\
   \boldsymbol{\mu} + \sqrt{(3 + \lambda) \boldsymbol{\Sigma}_{[i]}} & \text{for } i = 1, 2, 3 \\
   \boldsymbol{\mu} - \sqrt{(3 + \lambda) \boldsymbol{\Sigma}_{[i-3]}} & \text{for } i = 4, 5, 6
   \end{cases}
   $$
   Here, $\boldsymbol{\Sigma}_{[i]}$ denotes the $i$-th diagonal element of the covariance matrix $\boldsymbol{\Sigma}$, obtained via factorization [S1].

3. **Compute Weights:**
   Calculate weights for the mean ($w_i^\mu$) and covariance ($w_i^\Sigma$) [S1]:
   $$
   w_i^\mu = \begin{cases} 
   \frac{\lambda}{3+\lambda} & \text{for } i = 0 \\
   \frac{1}{2(3+\lambda)} & \text{for } i = 1, \dots, 6
   \end{cases}
   $$
   $$
   w_i^\Sigma = \begin{cases} 
   \frac{\lambda}{3+\lambda} + (1 - \alpha^2 + \beta) & \text{for } i = 0 \\
   \frac{1}{2(3+\lambda)} & \text{for } i = 1, \dots, 6
   \end{cases}
   $$
   [S1].

4. **Project Sigma Points:**
   Project each sigma point $\mathbf{x}_i$ independently onto the camera image plane using the nonlinear projection function $\mathbf{v}_{\mathbf{x}_i} = g(\mathbf{x}_i)$, which incorporates camera extrinsics $\mathbf{W}$ and intrinsics $\mathbf{D}$ [S1]. This step allows exact handling of distortions and time-dependent effects like rolling shutter by applying different extrinsic matrices if necessary [S1].

5. **Estimate 2D Mean:**
   Compute the projected 2D mean $\mathbf{v}_{\boldsymbol{\mu}}$ as the weighted posterior sample mean [S1]:
   $$
   \mathbf{v}_{\boldsymbol{\mu}} = \sum_{i=0}^6 w_i^\mu \mathbf{v}_{\mathbf{x}_i}
   $$
   [S1].

6. **Estimate 2D Covariance:**
   Compute the projected 2D covariance $\boldsymbol{\Sigma}'$ as the weighted posterior sample covariance [S1]:
   $$
   \boldsymbol{\Sigma}' = \sum_{i=0}^6 w_i^\Sigma (\mathbf{v}_{\mathbf{x}_i} - \mathbf{v}_{\boldsymbol{\mu}})(\mathbf{v}_{\mathbf{x}_i} - \mathbf{v}_{\boldsymbol{\mu}})^T
   $$
   [S1].

7. **Compute Extent and Rectangle:**
   Calculate the 2D extent $h_i$ using the estimated covariance $\boldsymbol{\Sigma}'_i$ and opacity $\sigma_i$ to determine a tighter bounding box [S1]. Compute the 2D rectangle $\mathbf{r}_i$ for tile-based rasterization culling [S1].

8. **Rasterization and Evaluation:**
   Use the computed 2D conic $(\mathbf{v}_{\boldsymbol{\mu}_i}, \boldsymbol{\Sigma}'_i)$ to determine which particles influence which pixels via tiling and culling [S1]. Note that the particle response evaluation itself does not depend on the 2D conic; instead, it evaluates the Gaussian particle response in 3D space and sorts particles to align with ray-tracing formulations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_2d_gaussian(mu, R, S, W, D, alpha=1.0, beta=2.0, kappa=0.0):
    # Step 1: Compute lambda
    lambda_val = alpha**2 * (3 + kappa) - 3
    
    # Step 2: Sample Sigma Points
    # Covariance factorization: Sigma = R @ S @ S.T @ R.T
    # Matrix square root L = R @ S
    L = [[R[i][j] * S[j][i] for j in range(3)] for i in range(3)]
    sigma_points = [[0.0, 0.0, 0.0] for _ in range(7)]
    sigma_points[0] = list(mu)
    for i in range(3):
        col_i = [L[j][i] for j in range(3)]
        scale = (3 + lambda_val)
        sqrt_scale = scale ** 0.5
        sigma_points[i+1] = [mu[j] + (col_i[j] * sqrt_scale) for j in range(3)]
        sigma_points[i+4] = [mu[j] - (col_i[j] * sqrt_scale) for j in range(3)]
        
    # Step 3: Compute Weights
    weights_mu = [0.0] * 7
    weights_sigma = [0.0] * 7
    weights_mu[0] = lambda_val / (3 + lambda_val)
    for i in range(1, 7):
        weights_mu[i] = 1.0 / (2 * (3 + lambda_val))
        
    weights_sigma[0] = lambda_val / (3 + lambda_val) + (1 - alpha**2 + beta)
    for i in range(1, 7):
        weights_sigma[i] = 1.0 / (2 * (3 + lambda_val))
        
    # Step 4: Project Points
    projected_points = [[0.0, 0.0] for _ in range(7)]
    for i in range(7):
        projected_points[i] = project_point(sigma_points[i], W, D)
        
    # Step 5: Estimate Mean
    v_mu = [0.0, 0.0]
    for i in range(7):
        v_mu[0] += weights_mu[i] * projected_points[i][0]
        v_mu[1] += weights_mu[i] * projected_points[i][1]
        
    # Step 6: Estimate Covariance
    Sigma_prime = [[0.0, 0.0], [0.0, 0.0]]
    for i in range(7):
        diff = [projected_points[i][0] - v_mu[0], projected_points[i][1] - v_mu[1]]
        for r in range(2):
            for c in range(2):
                Sigma_prime[r][c] += weights_sigma[i] * diff[r] * diff[c]
                
    return v_mu, Sigma_prime

def project_point(x_3d, W, D):
    # Transform to camera coordinates
    x_cam = [0.0, 0.0, 0.0]
    for i in range(3):
        x_cam[i] = W[i][0]*x_3d[0] + W[i][1]*x_3d[1] + W[i][2]*x_3d[2] + W[i][3]
        
    # Project to normalized plane
    x_norm = [x_cam[0]/x_cam[2], x_cam[1]/x_cam[2], 1.0]
    
    # Apply intrinsics
    x_2d = [0.0, 0.0]
    x_2d[0] = D[0][0]*x_norm[0] + D[0][1]*x_norm[1] + D[0][2]
    x_2d[1] = D[1][0]*x_norm[0] + D[1][1]*x_norm[1] + D[1][2]
    
    return x_2d
```

**Edge Cases and Pitfalls:**
- **Precision:** The UT avoids the linearization errors inherent in EWA splatting, which increase with camera distortion [S1].
- **Boundaries:** The method relies on the accuracy of the projection function $g(\mathbf{x})$. If $g$ is undefined for certain sigma points (e.g., behind the camera), handling is required [G].
- **Degenerate Input:** If the covariance matrix is singular, the square root factorization may fail [G].

## Subtopics

### [eval3d](15209-eval3d.md)
A rendering evaluation method for 3D Gaussian particles that computes the Gaussian response along the pixel's view ray rather than using the projected 2D footprint. It solves the visual degradation (washed-out, hazy appearance) that occurs when scenes trained with 3DGUT are rendered using standard EWA splatting. It is distinct from 3DGUT in that it implements the specific ray-based evaluation metric required by 3DGUT-trained data, whereas 3DGUT provides the training and projection framework.

## Operating characteristics
- **Time Complexity:** The UT projection requires computing 7 sigma points, projecting them, and computing weighted sums [S1]. This is a constant number of operations per particle relative to the linearization approach, but involves more arithmetic than a single Jacobian multiplication [S1].
- **Memory Complexity:** Requires storing 7 sigma points and weights per particle during projection [S1].
- **Measured Performance:** On standard novel-view synthesis benchmarks, 3DGUT supports distorted cameras [S1]. Performance varies with the degree of the Gaussian representation; for example, using degree 2 yields ~207 FPS, while degree 4 yields ~233 FPS on the tested hardware [S1].
- **Strengths:** Exact projection under arbitrary nonlinear camera models (distortion, rolling shutter); derivative-free; no need to derive Jacobians for different camera models; aligns with ray-tracing for secondary rays [S1].
- **Limitations:** Slightly more computationally intensive per particle than linearized EWA due to the 7-point projection; requires careful handling of the projection function $g(\mathbf{x})$ [S1].
- **When to use:** When rendering scenes with distorted cameras, rolling shutter effects, or when secondary ray tracing (reflections/refractions) is required [S1].

## Background
3D Gaussian Unscented Transform (3DGUT) was proposed in the paper "3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting" [S1]. It draws inspiration from the classical literature of the Unscented Kalman Filter [S1] and the Unscented Transform [S1]. The method addresses the limitations of EWA splatting in 3D Gaussian Splatting, which suffers from approximation errors with increasing distortion and cannot represent time-dependent effects like rolling shutter [S1]. It aligns the rasterization formulation with ray-tracing methods to enable secondary rays [S1].

## Sources
[S1] 3DGUT: Enabling Distorted Cameras and Secondary Rays in Gaussian Splatting, https://doi.org/10.1109/cvpr52734.2025.02425, fetched 2026-10-26
