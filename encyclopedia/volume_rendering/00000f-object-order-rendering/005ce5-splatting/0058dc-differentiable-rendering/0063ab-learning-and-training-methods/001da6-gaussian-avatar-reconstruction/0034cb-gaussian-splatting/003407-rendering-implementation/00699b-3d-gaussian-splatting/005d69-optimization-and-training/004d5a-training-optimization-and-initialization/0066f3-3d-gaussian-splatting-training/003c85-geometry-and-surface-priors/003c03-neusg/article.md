# NeuSG


## Overview
NeuSG [S1] integrates the NeuS implicit surface reconstruction method into the 3D Gaussian Splatting (3DGS) representation to transfer explicit surface properties to the discrete Gaussian primitives [S1]. Unlike sibling methods that rely on monocular depth priors or post-hoc mesh extraction, NeuSG directly couples a NeuS-style Signed Distance Function (SDF) with a Gaussian rasterizer [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Calibrated images, initial 3D Gaussian parameters (centers $\mathbf{c}_i$, covariances $\Sigma_i$, opacities $\alpha_i$, colors), and a NeuS network initialized to predict SDF and opacity [G].
*   **Outputs:** Optimized 3D Gaussians and a NeuS network that jointly render photorealistic images and a consistent surface [S1].

**Data Structures:**
1.  **3D Gaussians:** Standard 3DGS representation with centers $\mathbf{c}_i$, covariances $\Sigma_i$, opacities $\alpha_i$, and spherical harmonics coefficients for color [G].
2.  **NeuS Network:** An MLP taking 3D position $\mathbf{x}$ as input, outputting a raw SDF value $\sigma(\mathbf{x})$ and an opacity weight $\omega(\mathbf{x})$ [G].

**Numbered Steps:**
1.  **Initialization:** Initialize 3D Gaussians from sparse points and initialize the NeuS MLP [G].
2.  **Rendering:** Render the scene using the 3DGS rasterizer to produce RGB images [G].
3.  **Geometric Loss ($\mathcal{L}_{geo}$):** Query the NeuS network for the signed distance value at each Gaussian center: $d_i = \sigma(\mathbf{c}_i)$. Encourage these values to be zero, ensuring Gaussian centers lie on the zero-level set of the SDF [S1].
    $$ \mathcal{L}_{geo} = \sum_i |d_i| $$
4.  **Normal Consistency Loss ($\mathcal{L}_{norm}$):** Compute the Gaussian normal $\mathbf{n}_{gs,i}$ from its covariance matrix (principal axis). Compute the NeuS surface normal $\mathbf{n}_{neuS,i} = \nabla \sigma(\mathbf{c}_i) / \|\nabla \sigma(\mathbf{c}_i)\|$. Minimize the angular difference between them [S1].
    $$ \mathcal{L}_{norm} = \sum_i (1 - \mathbf{n}_{gs,i} \cdot \mathbf{n}_{neuS,i}) $$
5.  **Photometric Loss:** Compute the standard L1 or SSIM loss between rendered RGB images and ground truth images [G].
6.  **Optimization:** Minimize the total loss:
    $$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{geo} \mathcal{L}_{geo} + \lambda_{norm} \mathcal{L}_{norm} $$
    where $\lambda_{geo}$ and $\lambda_{norm}$ balance photometric fidelity against geometric accuracy [9270-geometry-and-surface-priors.md].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize(v):
    return v / max(norm(v), 1e-8)

def compute_gaussian_normal(cov):
    # Principal axis of the covariance matrix
    eigenvalues, eigenvectors = eig(cov)
    return normalize(eigenvectors[:, argmax(eigenvalues)])

def neusg_step(gaussians, neus_mlp, gt_rgb, lambda_geo, lambda_norm):
    # 1. Render RGB via 3DGS rasterizer (standard alpha compositing)
    rendered_rgb = rasterize(gaussians)
    
    # 2. Compute geometric and normal losses
    l_geo = 0.0
    l_norm = 0.0
    for g in gaussians:
        # Query NeuS SDF at Gaussian center
        d = neus_mlp(g.position)
        l_geo += abs(d)
        
        # Compute NeuS normal via analytical gradient of SDF
        n_neus = normalize(gradient(neus_mlp, g.position))
        
        # Compute Gaussian normal from covariance
        n_gs = compute_gaussian_normal(g.covariance)
        
        # Normal consistency
        l_norm += 1.0 - dot(n_gs, n_neus)
        
    l_geo /= len(gaussians)
    l_norm /= len(gaussians)
    
    # 3. Photometric loss
    l_rgb = mean(abs(rendered_rgb - gt_rgb))
    
    # 4. Total loss
    total_loss = l_rgb + lambda_geo * l_geo + lambda_norm * l_norm
    return total_loss
```
*Note: `rasterize`, `gradient`, `norm`, `dot`, `eig`, `argmax`, `abs`, `mean` are standard math/vector operations.*

**Parameters:**
*   $\lambda_{geo}$: Weight for geometric loss. Not given in the sources.
*   $\lambda_{norm}$: Weight for normal consistency loss. Not given in the sources.

**Edge Cases:**
*   **Precision:** SDF values near zero are sensitive to initialization [G]. Ensure NeuS is initialized to predict a large positive SDF outside the object and negative inside to avoid local minima.
*   **Boundaries:** Gaussians far from the surface may have noisy SDF gradients [G]. Consider masking Gaussians with low opacity or high distance from the zero-level set.

## Subtopics

### GSDF
Enforces SDF consistency but extracts the surface from accumulated Gaussian opacity rather than a separate implicit network [S2, unofficial].

### Surface and Geometry Alignment
Aligns Gaussians with explicit mesh normals or depth maps, often using post-processing or differentiable mesh rendering [S1].

### Monocular Depth Scale Alignment
Aligns 3DGS with monocular depth estimates without relying on an explicit SDF network [S1].

### Sparse-view 3D Gaussian Splatting with Depth Priors
Uses depth priors to regularize geometry in sparse views, potentially using SDF but not necessarily NeuS [S1].

## Operating characteristics
*   **Time Complexity:** Higher than vanilla 3DGS due to the additional NeuS network forward and backward passes for every Gaussian or ray sample [G]. Not given in the sources.
*   **Memory Complexity:** Increased by the storage of the NeuS MLP weights and intermediate SDF values [G]. Not given in the sources.
*   **Strengths:** Produces clean, watertight surfaces suitable for mesh extraction. Aligns Gaussian orientation with the true surface normal [S1].
*   **Limitations:** Slower training due to the coupled optimization. Requires careful tuning of $\lambda_{geo}$ and $\lambda_{norm}$ [G].
*   **When to use:** When a high-quality mesh is required for downstream applications (e.g., physics, DCC) [S2, unofficial].
*   **When not to use:** When real-time rendering speed is the primary concern and surface fidelity is less critical than appearance [S2, unofficial].

## Background
NeuSG incorporates the NeuS implicit surface reconstruction method into the 3DGS representation to transfer surface properties [S1]. It addresses the limitation of vanilla 3DGS, which lacks explicit geometric constraints, by enforcing positional and normal consistency with the implicit surface [9270-geometry-and-surface-priors.md]. The method is cited in surveys of recent advances in 3D Gaussian Splatting [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
