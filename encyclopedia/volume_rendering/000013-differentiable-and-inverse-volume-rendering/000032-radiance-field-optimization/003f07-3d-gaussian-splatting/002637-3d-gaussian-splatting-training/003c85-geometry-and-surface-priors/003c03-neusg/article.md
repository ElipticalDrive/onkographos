# NeuSG

Also called: NeuS-Gaussian Splatting

## Overview
NeuSG integrates the NeuS implicit surface reconstruction framework into the 3D Gaussian Splatting (3DGS) representation to enforce surface-aligned geometry [S1]. Unlike vanilla 3DGS, which produces floating ellipsoids, NeuSG transfers the continuous surface properties of NeuS to the discrete Gaussian primitives [S1]. It is distinguished from siblings like GSDF by its specific reliance on the NeuS SDF formulation rather than a generic SDF consistency loss [S1].

## How it works
**Inputs:** Camera poses, intrinsics, and input images for a set of views.
**Outputs:** A set of 3D Gaussians (position $\mathbf{p}$, covariance $\Sigma$, opacity $\alpha$, color $\mathbf{c}$) and an implicit SDF field $S(\mathbf{x})$ parameterized by a neural network.

**Data Structures:**
1.  **3D Gaussians:** Standard 3DGS representation. Each Gaussian $i$ has center $\mathbf{p}_i$, rotation matrix $\mathbf{R}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonics coefficients for color.
2.  **SDF Network:** A multi-layer perceptron (MLP) taking 3D coordinates $\mathbf{x}$ as input, outputting a scalar signed distance value $S(\mathbf{x})$ and optionally a normal vector. This is the NeuS network [S1].

**Steps:**
1.  **SDF Sampling:** For each Gaussian center $\mathbf{p}_i$, query the SDF network to obtain $S(\mathbf{p}_i)$ [S1].
2.  **Normal Estimation:** Compute the normal $\mathbf{n}_{GS}$ for each Gaussian from its covariance matrix (eigenvectors) or from the rendered normal map. Simultaneously, compute the normal $\mathbf{n}_{SDF}$ from the gradient of the SDF network at $\mathbf{p}_i$: $\mathbf{n}_{SDF} = \nabla S(\mathbf{p}_i) / \|\nabla S(\mathbf{p}_i)\|$ [S1].
3.  **Rendering:** Render the image using the standard 3DGS rasterizer, accumulating opacity and color [S2, unofficial].
4.  **Loss Calculation:**
    *   **Photometric Loss:** Standard $L_1$ or SSIM between rendered and ground truth images.
    *   **SDF Consistency Loss:** Encourage $S(\mathbf{p}_i) \approx 0$ for all Gaussians, aligning their centers with the implicit surface [S1].
    *   **Normal Consistency Loss:** Minimize the angular difference between $\mathbf{n}_{GS}$ and $\mathbf{n}_{SDF}$ [S1].
5.  **Optimization:** Jointly optimize Gaussian parameters and SDF network weights using gradient descent [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_step(gaussians, sdf_network, camera, image_gt):
    # 1. Query SDF at Gaussian centers
    sdf_values = sdf_network.forward(gaussians.positions)
    
    # 2. Compute SDF normals via finite differences or autograd
    sdf_normals = compute_sdf_normals(sdf_network, gaussians.positions)
    
    # 3. Compute Gaussian normals from covariance
    gs_normals = compute_gaussian_normals(gaussians.covariances)
    
    # 4. Render image
    image_pred = render_gaussians(gaussians, camera)
    
    # 5. Calculate losses
    # Photometric loss
    photometric_loss = l1_loss(image_pred, image_gt)
    
    # SDF consistency: encourage centers to lie on zero-level set
    sdf_loss = mean_squared_error(sdf_values, torch.zeros_like(sdf_values))
    
    # Normal consistency: align Gaussian orientation with SDF gradient
    # Dot product should be close to 1
    normal_dot = torch.sum(gs_normals * sdf_normals, dim=-1)
    normal_loss = 1.0 - normal_dot.mean()
    
    total_loss = photometric_loss + lambda_sdf * sdf_loss + lambda_normal * normal_loss
    
    # 6. Backpropagate
    total_loss.backward()
    optimizer.step()
```

**Parameters:**
*   $\lambda_{sdf}$: Weight for SDF consistency loss. Not given in the sources.
*   $\lambda_{normal}$: Weight for normal consistency loss. Not given in the sources.

**Edge Cases:**
*   If the SDF network is not well-initialized, $S(\mathbf{p}_i)$ may be large, causing instability in the consistency loss [S1].
*   Degenerate Gaussians (zero scale) may produce undefined normals [G].

## Subtopics

### GSDF
Uses a generic SDF consistency loss rather than the specific NeuS formulation. Choose when a simpler SDF integration is preferred over NeuS-specific priors. [GSDF](9205-gsdf.md)

### Surface and Geometry Alignment
A broader category including NeuSG. Choose when general surface alignment techniques are needed. [Surface and Geometry Alignment](9267-surface-and-geometry-alignment.md)

## Operating characteristics
*   **Time Complexity:** Higher than vanilla 3DGS due to the additional SDF network forward/backward passes for every Gaussian center per step [S1].
*   **Memory Complexity:** Increased by the storage of the SDF network weights [S1].
*   **Strengths:** Produces cleaner, surface-aligned geometry suitable for mesh extraction [S1].
*   **Limitations:** Slightly slower training than vanilla 3DGS; requires careful tuning of loss weights [S1].
*   **When to use:** When high-quality surface geometry is required for downstream tasks like meshing or physics [S1].
*   **When not to use:** When only photometric accuracy is needed and surface geometry is irrelevant [S1].

## Background
NeuSG was proposed to address the lack of explicit surface definition in 3DGS by incorporating the NeuS method [S1]. It builds on the success of NeuS in NeRF-based surface reconstruction [S1]. It is used in applications requiring high-fidelity 3D surface models from images [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
