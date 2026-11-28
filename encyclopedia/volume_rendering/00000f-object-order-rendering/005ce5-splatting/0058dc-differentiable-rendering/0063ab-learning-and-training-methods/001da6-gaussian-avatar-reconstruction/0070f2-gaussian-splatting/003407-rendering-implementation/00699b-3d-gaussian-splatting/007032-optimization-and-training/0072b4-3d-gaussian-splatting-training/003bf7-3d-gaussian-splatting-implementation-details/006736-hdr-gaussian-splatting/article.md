# HDR Gaussian Splatting

Also called: HDR-GS, High Dynamic Range Gaussian Splatting

## Overview
HDR Gaussian Splatting (HDR-GS) is a framework for 3D High Dynamic Range (HDR) imaging that renders novel HDR views and reconstructs Low Dynamic Range (LDR) images with user-controllable exposure time [S1]. It solves the problem of long training times and slow inference speeds associated with existing NeRF-based HDR Novel View Synthesis (NVS) methods [S1]. It sets itself apart by employing a Dual Dynamic Range (DDR) Gaussian point cloud model and two Parallel Differentiable Rasterization (PDR) processes to handle HDR and LDR rendering simultaneously [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of posed LDR images (typically from a dataset), camera poses (recalibrated from NDC to world coordinates), and an optional user-defined exposure time $\Delta t$ for LDR reconstruction [S1].
*   **Outputs:** A rendered HDR image and a rendered LDR image corresponding to the input camera pose [S1].

**Data Structures:**
The scene is represented by a set of $N_p$ Dual Dynamic Range (DDR) Gaussians $\mathcal{G} = \{G_i \mid i = 1, \dots, N_p\}$ [S1]. Each Gaussian $G_i$ contains the following attributes:
*   $\mu_i \in \mathbb{R}^3$: Center position.
*   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix.
*   $\alpha_i \in \mathbb{R}$: Opacity.
*   $c_{l,i} \in \mathbb{R}^3$: LDR color.
*   $c_{h,i} \in \mathbb{R}^3$: HDR color.
*   $\Delta t \in \mathbb{R}$: Exposure time (scalar controlling LDR intensity) [S1].
*   $\theta$: Global MLP parameters shared across all Gaussians for tone-mapping [S1].

The covariance matrix $\Sigma_i$ is decomposed into a rotation matrix $R_i \in \mathbb{R}^{3 \times 3}$ and a scaling matrix $S_i \in \mathbb{R}^{3 \times 3}$ as:
$$ \Sigma_i = R_i S_i S_i^\top R_i^\top $$
where $R_i$ and $S_i$ are learnable parameters [S1].

The HDR color $c_{h,i}$ is view-dependent and modeled using Spherical Harmonics (SH). Let $k = \{k_{lm} \mid 0 \le l \le L, -l \le m \le l\} \in \mathbb{R}^{(L+1)^2 \times 3}$ be the SH coefficients, where each $k_{lm} \in \mathbb{R}^3$ corresponds to RGB components. For a view direction $d = (\theta, \phi)$, the HDR color is:
$$ c_{h,i}(d, k) = \exp\left( \sum_{l=0}^{L} \sum_{m=-l}^{l} k_{lm} Y_{lm}(\theta, \phi) \right) $$
where $Y_{lm}: S^2 \to \mathbb{R}$ are the SH basis functions [S1].

**Steps:**
1.  **Initialization:** Recalibrate camera parameters from Normalized Device Coordinates (NDC) to world coordinates using Structure-from-Motion (SfM) to establish a suitable data foundation for 3DGS-based methods [S1]. Compute initial positions for the Gaussian point clouds [S1].
2.  **Tone-Mapping Formulation:** To avoid vanishing gradients and numerical overflow/underflow caused by direct multiplication in the linear domain, the tone-mapping function is transformed into the logarithmic domain [S1]. The relationship between LDR color $c_l$, HDR color $c_h$, and exposure time $\Delta t$ is defined as:
    $$ c_l = f_{TM}(c_h \cdot \Delta t) $$
    Inverting this and taking the natural logarithm ($\ln$, base $e$):
    $$ \log f_{TM}^{-1}(c_l) = \log c_l + \log \Delta t $$
    The inverse function is reformulated as:
    $$ c_l = (\log f_{TM}^{-1})^{-1}(\log c_h + \log \Delta t) $$
    [S1].
3.  **MLP Tone-Mapper:** Three independent MLPs with global parameters $\theta$ model the function $(\log f_{TM}^{-1})^{-1}$ independently for each RGB channel [S1]. Define the mapping function $g_\theta(x) \triangleq (\log f_{TM}^{-1})^{-1}(x)$. The LDR color is computed as:
    $$ c_{l,i} = g_\theta(\log c_{h,i} + \log \Delta t) $$
    [S1].
4.  **Parallel Differentiable Rasterization (PDR):** The HDR colors $c_{h,i}$ and LDR colors $c_{l,i}$ are fed into two separate Parallel Differentiable Rasterization processes to render the final HDR and LDR images, respectively [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spherical_harmonics_eval(direction_theta, direction_phi, coefficients, degree):
    """
    Evaluates Spherical Harmonics for HDR color.
    direction_theta: float, polar angle
    direction_phi: float, azimuthal angle
    coefficients: list of (l, m, k_lm) tuples where k_lm is np.array([R, G, B])
    degree: int, L
    Returns: np.array([R, G, B])
    """
    color = np.zeros(3)
    for l in range(degree + 1):
        for m in range(-l, l + 1):
            # Get k_lm for this l, m
            k_lm = None
            for (ll, mm, k_val) in coefficients:
                if ll == l and mm == m:
                    k_lm = k_val
                    break
            if k_lm is None:
                continue
            
            # Evaluate Y_lm(theta, phi)
            # Note: Standard SH basis functions Y_lm are used here.
            # Implementation details of Y_lm depend on the specific SH library or formula.
            y_lm = evaluate_sh_basis(l, m, direction_theta, direction_phi)
            
            color += k_lm * y_lm
            
    # Apply exponential as per Eq. 7
    return np.exp(color)

def tone_mapper_log_inverse(log_hdr_color, exposure_time, mlp_r, mlp_g, mlp_b):
    """
    Computes LDR color from HDR color and exposure time using learned MLPs.
    log_hdr_color: np.array([log_R, log_G, log_B])
    exposure_time: float
    mlp_r, mlp_g, mlp_b: MLP functions for R, G, B channels
    Returns: np.array([R, G, B]) in LDR range
    """
    log_exposure = np.log(exposure_time)
    
    # Input to MLPs is log(c_h) + log(delta_t)
    input_r = log_hdr_color[0] + log_exposure
    input_g = log_hdr_color[1] + log_exposure
    input_b = log_hdr_color[2] + log_exposure
    
    # Apply MLPs
    out_r = mlp_r(input_r)
    out_g = mlp_g(input_g)
    out_b = mlp_b(input_b)
    
    return np.array([out_r, out_g, out_b])

def render_hdr_gaussians(gaussians, camera_pose, sh_coefficients, degree):
    """
    Renders HDR image using Parallel Differentiable Rasterization.
    gaussians: list of Gaussian objects with attributes mu, Sigma, alpha, sh_coefficients
    camera_pose: Camera transformation matrix
    sh_coefficients: Global SH coefficients for each Gaussian
    degree: SH degree L
    Returns: HDR Image tensor
    """
    # 1. Project Gaussians to 2D
    projected_gaussians = []
    for g in gaussians:
        # Project center mu to camera space
        mu_cam = transform_point(g.mu, camera_pose)
        
        # Project covariance Sigma
        # J is Jacobian of projection, W is view transform
        # Sigma_2D = J W Sigma W^T J^T
        # (Implementation of 2D projection of 3D Gaussian)
        sigma_2d = project_covariance(g.Sigma, camera_pose)
        
        projected_gaussians.append({
            'mu_2d': project_2d(g.mu, camera_pose),
            'sigma_2d': sigma_2d,
            'alpha': g.alpha,
            'sh_coeffs': g.sh_coefficients
        })
        
    # 2. Rasterize
    # Sort by depth, iterate over pixels, accumulate splats
    # For each pixel, compute view direction d
    # Compute HDR color c_h using SH
    # Accumulate color weighted by alpha and Gaussian PDF
    pass

def render_ldr_gaussians(gaussians, camera_pose, exposure_time, mlp_r, mlp_g, mlp_b):
    """
    Renders LDR image using Parallel Differentiable Rasterization.
    gaussians: list of Gaussian objects
    camera_pose: Camera transformation matrix
    exposure_time: float
    mlp_r, mlp_g, mlp_b: Tone-mapping MLPs
    Returns: LDR Image tensor
    """
    # 1. Project Gaussians to 2D (Same as HDR)
    projected_gaussians = []
    for g in gaussians:
        mu_cam = transform_point(g.mu, camera_pose)
        sigma_2d = project_covariance(g.Sigma, camera_pose)
        
        # Compute HDR color c_h first (needed for tone mapping)
        # View direction d from camera to mu_cam
        d = normalize(mu_cam - camera_origin)
        theta, phi = spherical_coords(d)
        
        c_h = spherical_harmonics_eval(theta, phi, g.sh_coefficients, g.degree)
        
        # Compute LDR color c_l using tone mapper
        log_c_h = np.log(c_h + 1e-8) # Avoid log(0)
        c_l = tone_mapper_log_inverse(log_c_h, exposure_time, mlp_r, mlp_g, mlp_b)
        
        projected_gaussians.append({
            'mu_2d': project_2d(g.mu, camera_pose),
            'sigma_2d': sigma_2d,
            'alpha': g.alpha,
            'color': c_l
        })
        
    # 2. Rasterize LDR colors
    # Accumulate LDR color weighted by alpha and Gaussian PDF
    pass
```

**Parameters:**
*   $N_p$: The number of 3D Gaussians in the scene representation [S1].
*   $\Delta t$: The exposure time, a scalar value $\Delta t \in \mathbb{R}$ that controls the light intensity of the LDR view and can be read from photo metadata [S1].
*   $L$: The degree of Spherical Harmonics used for view-dependent HDR color [S1].
*   $\theta$: Global MLP parameters for the tone-mapper [S1].

**Edge Cases and Pitfalls:**
*   **Vanishing Gradients/Overflow:** Directly modeling the tone-mapping function $f_{TM}(\cdot)$ in the linear domain can cause vanishing gradients and numerical overflow/underflow due to multiplication operations [S1]. The logarithmic transformation is required to mitigate this [S1].
*   **NDC Coordinates:** Existing datasets often provide camera poses in Normalized Device Coordinates (NDC), which are not suitable for 3DGS-based methods. Recalibration to world coordinates is necessary [S1].
*   **RGB Channel Independence:** The tone-mapping function is modeled by three independent MLPs for RGB channels because RGB colors are tone-mapped by different Camera Response Functions (CRFs) [S1].

## Subtopics

### Dual Dynamic Range (DDR) Model
A Gaussian point cloud model that stores both HDR and LDR color attributes, enabling simultaneous rendering of both dynamic ranges [S1].

### Parallel Differentiable Rasterization (PDR)
A rendering technique that processes HDR and LDR colors through separate rasterization pipelines in parallel [S1].

### Logarithmic Tone-Mapping
A method to stabilize MLP training for tone-mapping by operating in the logarithmic domain to avoid numerical issues [S1].

## Operating characteristics
*   **Performance:** HDR-GS surpasses state-of-the-art NeRF-based methods by 3.84 dB on LDR Novel View Synthesis (NVS) and 1.91 dB on HDR NVS [S1].
*   **Speed:** It achieves 1000× inference speed compared to HDR-NeRF, which takes 8.2 seconds to infer an image at 400×400 resolution [S1].
*   **Training Time:** HDR-GS requires only 6.3% of the training time compared to HDR-NeRF, which takes 9 hours to train [S1].
*   **Strengths:** High inference speed, significantly reduced training time, and user-controllable exposure for LDR reconstruction [S1].
*   **Limitations:** Requires recalibration of camera parameters from NDC to world coordinates for initialization [S1].

## Background
HDR-GS was proposed to address the limitations of NeRF-based HDR NVS methods, which suffer from long training times and slow inference speeds [S1]. It builds on the 3D Gaussian Splatting framework [S1]. The method introduces the Dual Dynamic Range (DDR) Gaussian point cloud model and Parallel Differentiable Rasterization (PDR) processes [S1]. It also addresses the issue of NDC camera poses in existing datasets by recalibrating them using Structure-from-Motion (SfM) [S1].

## Sources
[S1] HDR-GS: Efficient High Dynamic Range Novel View Synthesis at 1000x Speed via Gaussian Splatting, https://arxiv.org/abs/2405.15125v4, fetched 2026-10-26
