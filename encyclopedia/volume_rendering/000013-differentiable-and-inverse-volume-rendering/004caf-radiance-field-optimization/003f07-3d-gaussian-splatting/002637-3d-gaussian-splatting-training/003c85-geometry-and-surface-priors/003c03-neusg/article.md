# NeuSG

Also called: NeuS-Gaussian Splatting, NeuS-3DGS

## Overview
NeuSG integrates the NeuS neural surface reconstruction method [S1] into the 3D Gaussian Splatting (3DGS) framework to transfer explicit surface properties to the discrete Gaussian representation [S1]. Unlike vanilla 3DGS, which produces floating ellipsoids as an emergent property, NeuSG enforces geometric consistency by coupling a continuous Signed Distance Function (SDF) with the Gaussian rasterizer [S2, unofficial]. It is chosen when high-fidelity mesh extraction is required directly from the Gaussian optimization process, distinguishing it from siblings like GSDF which may use different consistency mechanisms [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images, camera poses, initial 3D Gaussian points (position $\mathbf{c}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, color $\mathbf{c}_i$), and a neural SDF network $f_\theta(\mathbf{x})$ initialized from NeuS [S1].
*   **Outputs:** Optimized Gaussian parameters and SDF network weights, enabling surface-aligned rendering and mesh extraction [S1].

**Data Structures:**
*   **Gaussian Field:** List of $N$ Gaussians. Each Gaussian $i$ has position $\mathbf{c}_i \in \mathbb{R}^3$, rotation quaternion $\mathbf{q}_i$, scale vector $\mathbf{s}_i$, opacity $\alpha_i \in [0,1]$, and spherical harmonics coefficients for color.
*   **SDF Network:** A multi-layer perceptron $f_\theta: \mathbb{R}^3 \to \mathbb{R}$ taking 3D coordinates and outputting a signed distance value [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize 3D Gaussians from sparse point cloud (e.g., COLMAP). Initialize NeuS SDF network $f_\theta$ [S1].
2.  **Rendering:** Render images using the 3DGS rasterizer. Compute rendered depth $D_{render}$ and normals if supported by the rasterizer extension [S1].
3.  **SDF Evaluation:** For each Gaussian center $\mathbf{c}_i$, query the SDF network to get $d_i = f_\theta(\mathbf{c}_i)$ [S1].
4.  **Normal Computation:** Compute the geometric normal from the SDF gradient: $\mathbf{m}_i = \nabla f_\theta(\mathbf{c}_i) / \|\nabla f_\theta(\mathbf{c}_i)\|$ [S1]. Compute the Gaussian normal $\mathbf{n}_i$ from the covariance matrix eigenvectors or rotation quaternion [S1].
5.  **Loss Calculation:** Compute consistency losses (see Formulas).
6.  **Optimization:** Update Gaussian parameters and SDF weights $\theta$ jointly using gradient descent [S1].

**Formulas:**
*   **SDF Consistency:** Encourages Gaussian centers to lie on the zero-level set of the SDF [S1]:
    $$ \mathcal{L}_{sdf} = \frac{1}{N} \sum_{i=1}^{N} |f_\theta(\mathbf{c}_i)| $$
*   **Normal Consistency:** Aligns the Gaussian orientation with the SDF geometric normal [S1]:
    $$ \mathcal{L}_{normal} = \frac{1}{N} \sum_{i=1}^{N} (1 - \mathbf{n}_i \cdot \mathbf{m}_i) $$
    Where $\mathbf{n}_i$ is the Gaussian normal and $\mathbf{m}_i$ is the SDF normal [S1].

**Parameters:**
*   Loss weights for $\mathcal{L}_{sdf}$ and $\mathcal{L}_{normal}$: Not given in the sources.
*   SDF network architecture: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Gradient Flow:** The SDF gradient $\nabla f_\theta(\mathbf{c}_i)$ requires backpropagation through the neural network to the Gaussian positions. This can be unstable if the SDF is not well-initialized [S1].
*   **Degenerate Input:** If the initial point cloud is sparse or noisy, the SDF may fail to capture the surface, leading to poor normal consistency [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neusg_step(gaussians, sdf_network, images, cameras):
    # 1. Render current Gaussians
    rendered_images, rendered_depths = render_3dgs(gaussians, cameras)
    
    # 2. Compute SDF values and gradients for all Gaussian centers
    sdf_values = []
    sdf_normals = []
    for gaussian in gaussians:
        pos = gaussian.position
        # Query SDF
        d = sdf_network(pos)
        sdf_values.append(d)
        # Compute gradient of SDF at pos
        grad = compute_gradient(sdf_network, pos)
        # Normalize gradient to get normal
        normal = grad / norm(grad)
        sdf_normals.append(normal)
        
    # 3. Compute Gaussian normals from covariance/rotation
    gaussian_normals = [g.get_normal() for g in gaussians]
    
    # 4. Compute Losses
    # SDF Consistency: encourage |f(c)| -> 0
    l_sdf = mean([abs(d) for d in sdf_values])
    
    # Normal Consistency: align normals
    l_normal = mean([1 - dot(gn, sn) for gn, sn in zip(gaussian_normals, sdf_normals)])
    
    # 5. Backpropagate and update
    total_loss = l_sdf + l_normal # Weights not specified in sources
    total_loss.backward()
    optimizer.step()
    
    return rendered_images
```

## Subtopics

### NeuS Integration
The core mechanism of embedding the NeuS SDF into the 3DGS pipeline. It differs from [GSDF](9205-gsdf.md) by using the specific NeuS formulation for surface rendering. Choose when leveraging established NeuS priors is preferred.

### Surface Extraction
Extracting a mesh from the optimized Gaussians and SDF. It differs from [Surface and Geometry Alignment](9267-surface-and-geometry-alignment.md) by relying on the SDF zero-level set rather than post-hoc alignment. Choose when explicit mesh generation is the primary goal.

## Operating characteristics
*   **Time Complexity:** Higher than vanilla 3DGS due to the additional forward and backward passes through the SDF network for every Gaussian center per optimization step [S1].
*   **Memory Complexity:** Increased by the storage of the SDF network weights and the intermediate gradient computations [S1].
*   **Scaling:** Scales with the number of Gaussians $N$ and the complexity of the SDF network. Performance degrades if $N$ is very large due to the per-Gaussian SDF queries [S1].
*   **Strengths:** Produces surface-aligned Gaussians, facilitating direct mesh extraction and reducing floating artifacts [S1].
*   **Limitations:** Requires careful tuning of loss weights; SDF optimization can be unstable with poor initialization [S1].
*   **When to use:** When high-quality surface geometry is required for downstream tasks like physics or DCC pipelines [S1].
*   **When not to use:** When real-time rendering speed is the primary constraint and explicit surface geometry is not needed [S1].

## Background
NeuSG was introduced to address the lack of explicit surface geometry in vanilla 3D Gaussian Splatting [S1]. It builds on the NeuS method, which combines neural rendering with SDFs for surface reconstruction [S1]. The method was developed to transfer the surface properties of NeuS to the more efficient 3DGS representation [S1]. It is used in applications requiring high-fidelity 3D reconstruction with explicit mesh outputs [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
