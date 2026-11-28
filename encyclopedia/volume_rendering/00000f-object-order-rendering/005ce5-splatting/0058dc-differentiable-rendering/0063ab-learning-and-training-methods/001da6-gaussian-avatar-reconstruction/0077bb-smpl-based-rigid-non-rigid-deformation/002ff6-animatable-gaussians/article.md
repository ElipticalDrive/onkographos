# Animatable Gaussians
## Overview
Animatable Gaussians is a method for reconstructing animatable 3D human avatars using 3D Gaussian Splatting (3DGS) driven by the SMPL-X parametric model [S1, unofficial]. It solves the problem of generating high-fidelity, real-time renderable human avatars by explicitly modeling pose-conditioned deformation offsets [S1, unofficial]. It is distinguished from sibling methods like [3DGS-Avatar](1480-3dgs-avatar.md), which uses per-Gaussian deformation MLPs, and [Gaussian Avatar](1479-gaussian-avatar.md), which focuses on head-driven avatars using the FLAME model [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Inputs:** SMPL-X mesh parameters (pose $\theta$, expression $\beta$) [S1, unofficial].
*   **Outputs:** A set of 3D Gaussians with parameters (position, covariance, opacity, spherical harmonics coefficients) that can be rendered in real-time for any given pose/expression [S1, unofficial].

**Data Structures:**
Each Gaussian $i$ stores canonical position $\mathbf{p}_i$, initial covariance $\Sigma_i$, opacity $\alpha_i$, spherical harmonics coefficients $\mathbf{c}_i$, and explicit pose-conditioned offset parameters $\Delta \mathbf{p}$ [S1, unofficial].

**Steps:**
1.  **Initialization:** Fit SMPL-X to the input video to obtain the canonical mesh and initial parameters [S1, unofficial]. Initialize Gaussians on the canonical mesh surface.
2.  **Deformation:** For a target pose $\theta$ and expression $\beta$:
    *   Compute explicit pose-conditioned offset $\Delta \mathbf{p}(\theta, \beta)$ [S1, unofficial].
    *   Final position: $\mathbf{p}'_i = \mathbf{p}_i + \Delta \mathbf{p}(\theta, \beta)$ [S1, unofficial].
    *   Update covariance $\Sigma'_i$ using the Jacobian $J$ of the deformation map: $\Sigma'_i = J \Sigma_i J^T + \sigma^2 I$ [G].
3.  **Rendering:** Render the deformed Gaussians using differentiable 3D Gaussian Splatting [S1, unofficial].
4.  **Training:** Minimize photometric loss between rendered image and ground truth video frame [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_animatable_gaussians(gaussians, pose, expression):
    deformed_gaussians = []
    for g in gaussians:
        # 1. Compute explicit pose-conditioned offset
        delta_p = evaluate_offset(g['offset_params'], pose, expression)
        
        # 2. Final position
        p_final = [g['p'][k] + delta_p[k] for k in range(3)]
        
        # 3. Covariance update via Jacobian
        J = compute_jacobian(g['offset_params'], pose, expression, g['p'])
        cov_final = J @ g['cov'] @ J.T + 1e-4 * [[1,0,0],[0,1,0],[0,0,1]]
        
        deformed_gaussians.append({'p': p_final, 'cov': cov_final, 'alpha': g['alpha'], 'sh': g['sh']})
    
    # 4. Differentiable rasterization
    image = standard_3dgs_rasterize(deformed_gaussians)
    return image

def evaluate_offset(offset_params, pose, expression):
    """Returns explicit pose-conditioned offset. Architecture not specified in sources [S1, unofficial]."""
    return offset_params # Simplified representation

def compute_jacobian(offset_params, pose, expression, p):
    """Computes Jacobian of deformation map. Not specified in sources [G]."""
    eps = 1e-5
    J = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0], [0.0, 0.0, 0.0]]
    for i in range(3):
        p_plus = p[:]
        p_plus[i] += eps
        delta_plus = evaluate_offset(offset_params, pose, expression)
        delta_base = evaluate_offset(offset_params, pose, expression)
        J[i][i] = (delta_plus[i] - delta_base[i]) / eps
    return J

def standard_3dgs_rasterize(gaussians):
    """Projects Gaussians to screen, sorts by depth, and composites colors. Standard 3DGS step [G]."""
    return [[0.0, 0.0, 0.0]]
```

**Parameters:**
*   **Offset Parameters:** Not given in the sources. [S1, unofficial]
*   **Regularization $\sigma^2$:** Small value (e.g., $10^{-4}$) to ensure positive definiteness of covariance [G].

**Edge Cases:**
*   **Degenerate Poses:** Extreme poses may cause deformation artifacts; the explicit offset helps mitigate this but requires careful training data coverage [S1, unofficial].
*   **Precision:** Jacobian computation must be numerically stable [G].
## Subtopics

### [GSMs](7832-gsms.md)
Gaussian Shell Maps (GSMs) is a method for generating animatable 3D humans by combining CNN-based generators with 3D Gaussian Splatting (3DGS) to recreate virtual humans with sophisticated details such as clothing and accessories. It builds multi-layer shells from the SMPL template and binds 3D Gaussians to these shells. By utilizing the differentiable rendering of 3DGS and the generative adversarial network of StyleGAN2, animatable 3D humans are efficiently generated. It is distinguished from sibling methods like GAvatar, which attaches Gaussians to SMPL-X primitives, and HumanGaussian, which initializes Gaussians via random sampling.

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ where $N$ is the number of Gaussians, typical for 3DGS [G]. Deformation computation complexity is Not given in the sources.
*   **Memory Complexity:** $O(N)$ for storing Gaussian parameters and deformation offsets [G].
*   **Performance:** Real-time rendering is achievable on modern GPUs [G].
*   **Strengths:** High fidelity, real-time performance, explicit control via SMPL-X [S1, unofficial].
*   **Limitations:** Requires fitted SMPL-X parameters; quality depends on deformation offset modeling [S1, unofficial].
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.
## Background
Animatable Gaussians is an avatar method using the SMPL-X rig with explicit pose-conditioned offsets [S1, unofficial]. It builds on the SMPL-based rigid/non-rigid deformation principles [8385-smpl-based-rigid-non-rigid-deformation.md].
## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
