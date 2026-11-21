# Non-Linear Sphere Tracing

Also called: Non-linear sphere tracing for deformed SDFs, RaymarchNonlinear, Rendering deformed signed distance fields

## Overview
Non-Linear Sphere Tracing (NLST) is a ray marching method for rendering Signed Distance Fields (SDFs) that have been subjected to non-linear spatial deformations [S1]. It solves the problem that standard sphere tracing fails on deformed SDFs because the deformation invalidates the Euclidean distance metric required for conservative stepping, and computing the inverse deformation $D^{-1}$ along the ray is often impossible or expensive [S1]. Unlike standard sphere tracing, which steps linearly in world space, NLST traces rays in the undeformed (object) space by integrating an ordinary differential equation (ODE) that accounts for the deformation's Jacobian, allowing it to handle complex deformations like Linear Blend Skinning (LBS) and Free-Form Deformations (FFDs) without requiring analytic inverse transformations [S1].

## How it works
NLST reformulates the ray intersection problem as an initial value problem (IVP) in undeformed space [S1]. The core insight is that while a straight ray in deformed space becomes a curve in undeformed space, the arc-length parameterization of this curve allows the use of sphere tracing step sizes derived from the undeformed SDF [S1].

### Inputs and Outputs
*   **Inputs**: Ray origin $\mathbf{p}$ in deformed (world) space [S1]. Ray direction $\boldsymbol{\omega}$ in deformed space (normalized) [S1]. Deformation function $D: \mathbb{R}^3 \to \mathbb{R}^3$ mapping undeformed points to deformed points [S1]. Undeformed SDF $S: \mathbb{R}^3 \to \mathbb{R}$ [S1]. Maximum steps $N_{max}$, maximum distance $t_{max}$, convergence threshold $\epsilon$ [S2, unofficial].
*   **Outputs**: Hit distance $t$ in deformed space [S1]. Hit point $\mathbf{x}$ in deformed space [S1]. Normal $\mathbf{n}$ in deformed space [S1].

### Data Structures
*   **SceneHit**: A structure containing `position` (`float3`, deformed space), `normal` (`float3`, deformed space), `distanceTravelled` (`float`, accumulated distance), and `hit` (`bool`) [S2, unofficial].

### Algorithm Steps
1.  **Hull-Based Initialization**: To avoid computing $D^{-1}(\mathbf{p})$ for the ray origin, generate a coarse triangle mesh hull enclosing the implicit surface using marching cubes at a low resolution (e.g., $32^3$ grid) [S1]. Deform this hull using $D$ and rasterize it [S1]. For the current ray, retrieve the deformed-space position from the rasterized hull and transform it back to undeformed space via barycentric interpolation across the corresponding hull face [S1]. This provides an approximate undeformed start point $\hat{\mathbf{x}}_0$ close to the surface [S1]. If no hull is used, $\hat{\mathbf{x}}_0$ must be computed as $D^{-1}(\mathbf{p})$ [S1].
2.  **Ray Integral Formulation**: The ray in undeformed space $\hat{\mathbf{x}}(\hat{s})$ is parameterized by arc-length $\hat{s}$:
    $$ \hat{\mathbf{x}}(\hat{s}) = \hat{\mathbf{x}}_0 + \int_0^{\hat{s}} \hat{\boldsymbol{\omega}}(\hat{\mathbf{x}}(t)) \, dt $$
    where the unit tangent vector $\hat{\boldsymbol{\omega}}$ is:
    $$ \hat{\boldsymbol{\omega}}(\hat{\mathbf{x}}) = \frac{J_D^{-1}(\hat{\mathbf{x}}) \boldsymbol{\omega}}{\| J_D^{-1}(\hat{\mathbf{x}}) \boldsymbol{\omega} \|} $$
    Here, $J_D^{-1}(\hat{\mathbf{x}})$ is the inverse of the Jacobian matrix of the deformation $D$ evaluated at the undeformed point $\hat{\mathbf{x}}$ [S1].
3.  **Marching Loop**: For each step $i$ from 0 to $N_{max}-1$:
    *   **Evaluate SDF**: Compute $d = S(\hat{\mathbf{x}}_i)$ [S1].
    *   **Termination**: If $d < \epsilon$, a hit is detected [S1]. If accumulated distance $> t_{max}$, escape [S2, unofficial].
    *   **Step Size**: Let $h = |d|$ [S2, unofficial].
    *   **Integration**: Advance the position using a 4th-order Runge-Kutta integrator to find $\hat{\mathbf{x}}_{i+1}$ [S2, unofficial]. The vector field is $\boldsymbol{\omega}(\hat{\mathbf{x}}, \boldsymbol{\omega}_0) = \text{normalize}(J_D^{-1}(\hat{\mathbf{x}}) \boldsymbol{\omega}_0)$ [S2, unofficial].
        $$ \begin{aligned} k_1 &= \boldsymbol{\omega}(\hat{\mathbf{x}}_i, \boldsymbol{\omega}_0) \\ k_2 &= \boldsymbol{\omega}(\hat{\mathbf{x}}_i + 0.5 h k_1, \boldsymbol{\omega}_0) \\ k_3 &= \boldsymbol{\omega}(\hat{\mathbf{x}}_i + 0.5 h k_2, \boldsymbol{\omega}_0) \\ k_4 &= \boldsymbol{\omega}(\hat{\mathbf{x}}_i + h k_3, \boldsymbol{\omega}_0) \\ \hat{\mathbf{x}}_{i+1} &= \hat{\mathbf{x}}_i + \frac{h}{6} (k_1 + 2k_2 + 2k_3 + k_4) \end{aligned} $$
        This sub-stepping ensures accuracy along the curved ray path without requiring additional SDF evaluations [S1].
4.  **Hit Processing**:
    *   **Deformed Position**: Compute $\mathbf{x}_{hit} = D(\hat{\mathbf{x}}_{final})$ [S1].
    *   **Normal Transformation**: Retrieve the surface normal from the SDF via finite differences $\mathbf{n}_{obj} = \nabla S(\hat{\mathbf{x}}_{final})$ [S1]. Transform it to deformed space:
        $$ \mathbf{n}_{deformed} = \frac{(J_D^{-1}(\hat{\mathbf{x}}_{final}))^T \mathbf{n}_{obj}}{\| (J_D^{-1}(\hat{\mathbf{x}}_{final}))^T \mathbf{n}_{obj} \|} $$
        [S1][S2, unofficial].
    *   **Distance**: The hit distance in deformed space is the accumulated arc-length or computed via ray parameterization [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize(v):
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if length == 0: return (0.0, 0.0, 0.0)
    return (v[0]/length, v[1]/length, v[2]/length)

def matrix_vector_mul(M, v):
    return (
        M[0][0]*v[0] + M[0][1]*v[1] + M[0][2]*v[2],
        M[1][0]*v[0] + M[1][1]*v[1] + M[1][2]*v[2],
        M[2][0]*v[0] + M[2][1]*v[1] + M[2][2]*v[2]
    )

def transpose_matrix(M):
    return (
        (M[0][0], M[1][0], M[2][0]),
        (M[0][1], M[1][1], M[2][1]),
        (M[0][2], M[1][2], M[2][2])
    )

def invert_matrix_3x3(M):
    # Standard 3x3 matrix inversion [G]
    inv_det = 1.0 / (M[0][0]*(M[1][1]*M[2][2] - M[1][2]*M[2][1]) - 
                     M[0][1]*(M[1][0]*M[2][2] - M[1][2]*M[2][0]) + 
                     M[0][2]*(M[1][0]*M[2][1] - M[1][1]*M[2][0]))
    return (
        ((M[1][1]*M[2][2] - M[1][2]*M[2][1])*inv_det, (M[0][2]*M[2][1] - M[0][1]*M[2][2])*inv_det, (M[0][1]*M[1][2] - M[0][2]*M[1][1])*inv_det),
        ((M[1][2]*M[2][0] - M[1][0]*M[2][2])*inv_det, (M[0][0]*M[2][2] - M[0][2]*M[2][0])*inv_det, (M[0][2]*M[1][0] - M[0][0]*M[1][2])*inv_det),
        ((M[1][0]*M[2][1] - M[1][1]*M[2][0])*inv_det, (M[0][1]*M[2][0] - M[0][0]*M[2][1])*inv_det, (M[0][0]*M[1][1] - M[0][1]*M[1][0])*inv_det)
    )

def compute_jacobian(deform_func, x, h=0.0001):
    # Central finite differences for deformation Jacobian [S2, unofficial]
    h_vec = (h, 0.0, 0.0)
    dfdx = tuple(deform_func((x[0]+h_vec[0], x[1]+h_vec[1], x[2]+h_vec[2]))[i] - deform_func((x[0]-h_vec[0], x[1]-h_vec[1], x[2]-h_vec[2]))[i] for i in range(3))
    h_vec = (0.0, h, 0.0)
    dfdy = tuple(deform_func((x[0]+h_vec[0], x[1]+h_vec[1], x[2]+h_vec[2]))[i] - deform_func((x[0]-h_vec[0], x[1]-h_vec[1], x[2]-h_vec[2]))[i] for i in range(3))
    h_vec = (0.0, 0.0, h)
    dfdz = tuple(deform_func((x[0]+h_vec[0], x[1]+h_vec[1], x[2]+h_vec[2]))[i] - deform_func((x[0]-h_vec[0], x[1]-h_vec[1], x[2]-h_vec[2]))[i] for i in range(3))
    return (
        (dfdx[0]/(2*h), dfdy[0]/(2*h), dfdz[0]/(2*h)),
        (dfdx[1]/(2*h), dfdy[1]/(2*h), dfdz[1]/(2*h)),
        (dfdx[2]/(2*h), dfdy[2]/(2*h), dfdz[2]/(2*h))
    )

def compute_sdf_gradient(sdf_func, x, h=0.0001):
    # Central finite differences for SDF gradient [G]
    h_vec = (h, 0.0, 0.0)
    dfdx = (sdf_func((x[0]+h_vec[0], x[1]+h_vec[1], x[2]+h_vec[2])) - sdf_func((x[0]-h_vec[0], x[1]-h_vec[1], x[2]-h_vec[2])))/(2*h)
    h_vec = (0.0, h, 0.0)
    dfdy = (sdf_func((x[0]+h_vec[0], x[1]+h_vec[1], x[2]+h_vec[2])) - sdf_func((x[0]-h_vec[0], x[1]-h_vec[1], x[2]-h_vec[2])))/(2*h)
    h_vec = (0.0, 0.0, h)
    dfdz = (sdf_func((x[0]+h_vec[0], x[1]+h_vec[1], x[2]+h_vec[2])) - sdf_func((x[0]-h_vec[0], x[1]-h_vec[1], x[2]-h_vec[2])))/(2*h)
    return (dfdx, dfdy, dfdz)

def nlst(ray_origin_deformed, ray_direction_deformed, deform_func, sdf_func, max_steps=200, max_dist=400.0, epsilon=0.0001):
    # 1. Initialization: x_hat_0 is typically obtained via hull-based initialization [S1]
    # If not, it requires D_inv(ray_origin_deformed) [S1]
    x_hat = ray_origin_deformed # Placeholder for hull interpolation or D_inv [G]
    omega_0 = ray_direction_deformed
    dist_travelled = 0.0
    
    for _ in range(max_steps):
        if dist_travelled > max_dist:
            return None, None, None
            
        sdf_val = sdf_func(x_hat)
        
        if sdf_val < epsilon:
            x_deformed = deform_func(x_hat)
            n_obj = compute_sdf_gradient(sdf_func, x_hat)
            J = compute_jacobian(deform_func, x_hat)
            J_inv = invert_matrix_3x3(J)
            J_T = transpose_matrix(J_inv)
            n_deformed_raw = matrix_vector_mul(J_T, n_obj)
            n_deformed = normalize(n_deformed_raw)
            return dist_travelled, x_deformed, n_deformed
            
        h = abs(sdf_val)
        
        def get_omega(x):
            J = compute_jacobian(deform_func, x)
            J_inv = invert_matrix_3x3(J)
            v = matrix_vector_mul(J_inv, omega_0)
            return normalize(v)
            
        k1 = get_omega(x_hat)
        k2 = get_omega((x_hat[0] + 0.5*h*k1[0], x_hat[1] + 0.5*h*k1[1], x_hat[2] + 0.5*h*k1[2]))
        k3 = get_omega((x_hat[0] + 0.5*h*k2[0], x_hat[1] + 0.5*h*k2[1], x_hat[2] + 0.5*h*k2[2]))
        k4 = get_omega((x_hat[0] + h*k3[0], x_hat[1] + h*k3[1], x_hat[2] + h*k3[2]))
        
        x_hat = (
            x_hat[0] + (h/6.0)*(k1[0] + 2*k2[0] + 2*k3[0] + k4[0]),
            x_hat[1] + (h/6.0)*(k1[1] + 2*k2[1] + 2*k3[1] + k4[1]),
            x_hat[2] + (h/6.0)*(k1[2] + 2*k2[2] + 2*k3[2] + k4[2])
        )
        dist_travelled += h
        
    return None, None, None
```

### Parameters
*   `MAX_STEPS`: Integer, default 200 [S2, unofficial].
*   `MAX_DISTANCE`: Float, default 400.0 [S2, unofficial].
*   `THRESHOLD` ($\epsilon$): Float, default 0.0001 [S2, unofficial].
*   `NORMAL_DIFFERENTIAL` ($h$): Float, default 0.0001, used for Jacobian and SDF gradient finite differences [S2, unofficial].
*   `DEFORMATION_PARAMETER` ($\alpha$): Float, default 1.0, used in specific deformation functions (e.g., sinusoidal twist) [S2, unofficial].
*   **Hull Resolution**: Typically low, rarely higher than $32^3$, used for initialization [S1].

### Edge Cases and Pitfalls
*   **Inverse Deformation**: The method requires $D^{-1}$ at least once to seed $\hat{\mathbf{x}}_0 = D^{-1}(\mathbf{p})$ if hull initialization is not used. This makes it incompatible with deformations where the inverse does not exist or is not unique (e.g., some smooth skinning cases) [S1]. Hull initialization avoids this [S1].
*   **Jacobian Singularity**: The Jacobian $J_D$ must be invertible. If the deformation causes folding or singularities, the inverse is unstable [S1][S2, unofficial].
*   **Concave Objects**: Limited max steps may cause premature termination on concave objects, missing the surface [S1].
*   **Performance**: SDF evaluations are expensive. Naive step reduction improves accuracy but degrades performance [S1]. The Runge-Kutta integration requires multiple Jacobian evaluations per step, which is computationally heavy [S2, unofficial].
*   **Gradient Bound**: Sphere tracing requires $\|\nabla S\| \approx 1$. If the gradient bound is violated, step sizes are no longer conservative [S1].

## Subtopics

### [Coarse-to-Fine Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md)
Coarse-to-Fine Sphere Tracing is an acceleration strategy that exploits spatial coherence by marching at low resolution first. It differs from NLST by operating in linear Euclidean space rather than deformed manifolds, serving as a general optimization for standard sphere tracing rather than a deformation-specific solver.

### [Parabolic Sphere Tracing](14525-parabolic-sphere-tracing.md)
Parabolic Sphere Tracing is a method for heterogeneous media where the squared refractive index varies linearly. It sets itself apart from NLST by replacing iterative distance-field stepping with closed-form polynomial evaluation, avoiding numerical integration (Runge-Kutta) in favor of exact algebraic solutions when the refractive index gradient is constant.

## Operating characteristics
*   **Time Complexity**: Dominated by SDF evaluations and Jacobian computations per step. Frame times on NVIDIA Titan RTX at 1080p are comparable to low-resolution marching cubes (e.g., 10-13ms vs 6-9ms for S=32-64) but significantly faster than high-resolution meshing (e.g., 150ms+ for S=256) [S1].
*   **Memory Complexity**: Low, as it does not require storing a dense mesh or voxel grid for the final render, only the SDF and deformation parameters [S1].
*   **Strengths**: Preserves thin/sharp features, supports infinite resolution, handles complex forward deformations (LBS, FFDs) without inverse computation (with hull init), and allows real-time editing [S1].
*   **Limitations**: Slower than rasterized explicit geometry, sensitive to Jacobian singularities, requires $\|\nabla S\| \approx 1$, and performance degrades at grazing angles [S1].
*   **When to Use**: Interactive rendering of deformed implicit surfaces, character animation with LBS, volumetric modeling with deformation [S1].
*   **When Not to Use**: When inverse deformation is available and simple, or when extreme performance is required and explicit meshes are acceptable [S1].

## Background
NLST was introduced by Dario Seyb, Alec Jacobson, Derek Nowrouzezahrai, and Wojciech Jarosz in 2020 [S1]. It builds on Barr's [1984] formulation for deforming parametric curves/surfaces [S1]. The method addresses the limitations of meshing-based deformation (loss of detail, recomputation cost) and inverse-deformation-based methods (non-existence of inverses) [S1]. It is implemented in prototype applications using Unity3D and supports various SDF representations including grids, procedural shaders, and analytic brushes [S1].

## Sources
[S1] Non-linear sphere tracing for rendering deformed signed distance fields, https://doi.org/10.1145/3355089.3356502, fetched 2026-10-29
[S2] AustinGoodman/non-linear-sphere-tracing, https://github.com/AustinGoodman/non-linear-sphere-tracing, fetched 2026-10-26 (unofficial)
