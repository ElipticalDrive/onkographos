# Total Loss Function for Neural Scene Reconstruction

Also called: Multi-component Training Loss, Neural Scene Reconstruction Loss

## Overview
This method defines the composite objective function used to optimize neural scene reconstruction models utilizing Signed Radial Distance Functions (SRDF) [S1]. It combines photometric, geometric, and regularization terms into a single scalar loss to support training [S1]. It is distinguished from photometric-only approaches by explicitly volume rendering 2.5D depth and normal maps from the density field $\sigma_{SRDF}$ to exploit geometric cues, and by incorporating 3D visibility priors that provide direct constraints on 3D geometry [S1].

## How it works
**Inputs:** Camera poses and intrinsics for a batch of views, ground truth RGB images, and neural network parameters defining the SRDF density field $\sigma_{SRDF}$ and color field [S1].
**Outputs:** Scalar total loss value $\mathcal{L}$ and gradients for backpropagation [G].
**Data Structures:** The loss components are computed per-pixel or per-ray and aggregated (summed or averaged) over the batch [G]. Weights $\lambda_k$ are scalar hyperparameters [G].

**Formulas:**
The total loss is defined as [S1]:
$$ \mathcal{L} = \mathcal{L}_c + \lambda_n \mathcal{L}_n + \lambda_d \mathcal{L}_d + \lambda_e \mathcal{L}_e + \lambda_s \mathcal{L}_s + \lambda_{con} \mathcal{L}_{con} + \lambda_{vis} \mathcal{L}_{vis} $$
Where:
*   $\mathcal{L}_c$: RGB photometric loss.
*   $\mathcal{L}_n$: Normal consistency loss.
*   $\mathcal{L}_d$: Depth consistency loss.
*   $\mathcal{L}_e$: Eikonal regularization loss.
*   $\mathcal{L}_s$: Smoothness regularization loss.
*   $\mathcal{L}_{con}$: SRDF-SDF consistency loss.
*   $\mathcal{L}_{vis}$: Visibility loss.
*   $\lambda_n, \lambda_d, \lambda_e, \lambda_s, \lambda_{con}, \lambda_{vis}$: Scalar hyperparameters controlling the weight of each term. Typical values are not specified in the source [S1].

**Steps:**
1. **Volume Rendering of Geometry:** Compute 2.5D depth and normal maps using volume rendering with the density function $\sigma_{SRDF}$ [S1]. This exploits geometric cues to support training [S1].
2. **Compute RGB Loss ($\mathcal{L}_c$):** Compare the rendered RGB image against the ground truth RGB image [S1]. The exact metric formulation is deferred to supplementary material [S1].
3. **Compute Normal Loss ($\mathcal{L}_n$):** Compute the normal loss component [S1]. The exact formulation is deferred to supplementary material [S1].
4. **Compute Depth Loss ($\mathcal{L}_d$):** Compare the rendered depth map against ground truth or consistency targets [S1]. Details are deferred to supplementary material [S1].
5. **Compute Eikonal Loss ($\mathcal{L}_e$):** Compute the Eikonal regularization loss component [S1]. The exact formulation is deferred to supplementary material [S1].
6. **Compute Smooth Loss ($\mathcal{L}_s$):** Compute the smoothness regularization loss component [S1]. The exact formulation is deferred to supplementary material [S1].
7. **Compute SRDF-SDF Consistency Loss ($\mathcal{L}_{con}$):** Compute the consistency loss between the SRDF and SDF representations [S1]. The exact formulation is deferred to supplementary material [S1].
8. **Compute Visibility Loss ($\mathcal{L}_{vis}$):** Apply a visibility prior that provides 3D space priors, imparting a more direct influence on 3D geometry than 2D supervision alone [S1]. The exact formulation is deferred to supplementary material [S1].
9. **Aggregate:** Sum the weighted components to produce the final loss $\mathcal{L}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_total_loss(rendered_rgb, gt_rgb, rendered_depth, rendered_normal,
                       depth_gt=None, normal_gt=None, network_params=None, camera_params=None,
                       lambda_n=1.0, lambda_d=1.0, lambda_e=1.0, lambda_s=1.0,
                       lambda_con=1.0, lambda_vis=1.0):
    # 1. RGB Loss: Compare rendered RGB against ground truth [S1].
    # Exact metric not given in the sources. Computed per-pixel and aggregated over batch [G].
    L_c = 0.0  # Not given in the sources. Computed per-pixel and aggregated over batch [G]

    # 2. Normal Loss: Compare rendered normal against depth-derived normal [S1].
    # Exact metric not given in the sources. Computed per-pixel and aggregated over batch [G].
    L_n = 0.0  # Not given in the sources. Computed per-pixel and aggregated over batch [G]

    # 3. Depth Loss: Compare rendered depth against ground truth [S1].
    # Exact metric not given in the sources. Computed per-pixel and aggregated over batch [G].
    L_d = 0.0  # Not given in the sources. Computed per-pixel and aggregated over batch [G]

    # 4. Eikonal Loss: Enforce Eikonal constraint on SDF gradient [S1].
    # Exact formulation not given in the sources. Computed per-pixel and aggregated over batch [G].
    L_e = 0.0  # Not given in the sources. Computed per-pixel and aggregated over batch [G]

    # 5. Smooth Loss: Regularize neural field smoothness [S1].
    # Exact formulation not given in the sources. Computed per-pixel and aggregated over batch [G].
    L_s = 0.0  # Not given in the sources. Computed per-pixel and aggregated over batch [G]

    # 6. SRDF-SDF Consistency Loss: Enforce SRDF-SDF consistency [S1].
    # Exact formulation not given in the sources. Computed per-pixel and aggregated over batch [G].
    L_con = 0.0  # Not given in the sources. Computed per-pixel and aggregated over batch [G]

    # 7. Visibility Loss: Apply 3D visibility prior [S1].
    # Exact formulation not given in the sources. Computed per-pixel and aggregated over batch [G].
    L_vis = 0.0  # Not given in the sources. Computed per-pixel and aggregated over batch [G]

    # Aggregate weighted components [S1]
    L_total = (L_c +
               lambda_n * L_n +
               lambda_d * L_d +
               lambda_e * L_e +
               lambda_s * L_s +
               lambda_con * L_con +
               lambda_vis * L_vis)
    return L_total
```

## Subtopics

### Gaussian Scene Loss
A variant used in Gaussian Splatting-based reconstruction (e.g., $\text{VG}^2$GT) [S3, unofficial]. It differs by using Gaussian primitives instead of a continuous density field [S3, unofficial]. The loss combines RGB, SSIM, depth, and normal terms [S3, unofficial].
**Formula:**
$$ \mathcal{L}_G = \mathcal{L}_{c1} + \mathcal{L}_{cSSIM} + \mathcal{L}_1(D, D^G) + \mathcal{L}_1(D_{gt}, D^G) + \mathcal{L}_n $$
Where:
*   $\mathcal{L}_{c1}$: L1 loss on RGB [S3, unofficial].
*   $\mathcal{L}_{cSSIM}$: SSIM loss on RGB [S3, unofficial].
*   $D$: Regressed depth [S3, unofficial].
*   $D^G$: Rendered depth [S3, unofficial].
*   $D_{gt}$: Ground truth depth [S3, unofficial].
*   $\mathcal{L}_n$: Normal loss, computed via cosine similarity between rendered normal map and normal map derived from depth [S3, unofficial].

## Operating characteristics
*   **Complexity:** The computational cost is dominated by the volume rendering steps for RGB, depth, and normals, which are performed per batch iteration [G]. The additional geometric losses require gradient computations through the neural network, adding overhead proportional to the network size [G].
*   **When to use:** Suitable for indoor scene reconstruction where high-fidelity 3D geometry is required [S1].

## Background
The method originates from research on Ray-Distance Volume Rendering for Neural Scene Reconstruction, specifically utilizing Signed Radial Distance Functions (SRDF) [S1]. It builds upon prior work such as MonoSDF and NeuRIS, adapting their loss structures to the SRDF framework [S1]. The approach was developed to address limitations in SDF-based volume rendering, particularly regarding the learning of SDF in complex spaces [S1].

## Sources
[S1] Ray-Distance Volume Rendering for Neural Scene Reconstruction, https://doi.org/10.1007/978-3-031-72630-9_22, fetched 2026-10-26
[S2] Inverse Rendering of Translucent Objects using Physical and Neural Renderers, https://doi.org/10.1109/cvpr52729.2023.01204, fetched 2026-10-26
[S3] $\text{VG}^2$GT: Voxel-Gaussian Splatting Visual Geometry Grounded Transformer, https://doi.org/10.48550/arxiv.2606.01573, fetched 2026-10-26 (unofficial)
