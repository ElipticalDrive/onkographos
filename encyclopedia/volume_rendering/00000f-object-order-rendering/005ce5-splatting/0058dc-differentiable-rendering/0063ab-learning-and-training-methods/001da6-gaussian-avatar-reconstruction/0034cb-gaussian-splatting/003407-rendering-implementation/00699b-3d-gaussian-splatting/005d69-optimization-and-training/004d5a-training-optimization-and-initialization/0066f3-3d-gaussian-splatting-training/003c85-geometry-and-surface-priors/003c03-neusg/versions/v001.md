# NeuSG

Also called: NeuS-Gaussian Splatting

## Overview
NeuSG is a method that incorporates the NeRF-based surface reconstruction technique NeuS into the 3D Gaussian Splatting (3DGS) representation to transfer explicit surface properties to the Gaussian primitives [S1]. It solves the problem of vanilla 3DGS producing "floating" ellipsoids rather than coherent surfaces by enforcing geometric consistency between the Gaussians and a learned Signed Distance Function (SDF) [S1][S2, unofficial]. Unlike siblings that rely on post-hoc mesh extraction or opacity level-sets, NeuSG integrates surface priors directly into the training loop [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussian primitives (position $\mathbf{p}$, covariance $\Sigma$, opacity $\alpha$, color $\mathbf{c}$), a camera pose, and an image.
*   **Outputs:** Rendered image, updated Gaussian parameters, and a consistent surface representation.

**Data Structures:**
*   **3DGS Primitives:** Standard 3DGS structure.
*   **SDF Network:** A multi-layer perceptron (MLP) predicting the signed distance value $d(\mathbf{x})$ and its gradient (normal) $\nabla d(\mathbf{x})$ at any 3D point $\mathbf{x}$, based on the NeuS architecture [S1].

**Steps:**
1.  **Render Depth and Normals:** Render the scene using the current 3DGS parameters. Simultaneously, query the SDF network for depth and normal information at relevant 3D locations [S1].
2.  **SDF Consistency Loss:** Encourage the signed distance values of the Gaussian centers to be zero. This aligns the Gaussians with the implicit surface defined by the SDF [S1].
    $$ \mathcal{L}_{sdf} = \sum_{i} |d(\mathbf{p}_i)| $$
    where $\mathbf{p}_i$ is the center of the $i$-th Gaussian and $d(\cdot)$ is the SDF value [S1].
3.  **Normal Consistency Loss:** Encourage the normal directions derived from the 3DGS (or the rendered normals) to be consistent with the normals predicted by the NeuS SDF network [S1].
    $$ \mathcal{L}_{normal} = \sum_{i} (1 - \mathbf{n}_{gs,i} \cdot \mathbf{n}_{sdf,i}) $$
    where $\mathbf{n}_{gs,i}$ is the normal associated with the Gaussian and $\mathbf{n}_{sdf,i} = \nabla d(\mathbf{p}_i)$ is the surface normal from the SDF [S1].
4.  **Depth Alignment:** Ensure the closest Gaussian to a pixel aligns with the rendered depth value, further enforcing alignment with the authentic 3D surface [S1].
5.  **Optimization:** Minimize the combined loss (image reconstruction + geometric losses) using gradient descent [S1].

**Parameters:**
*   Loss weights for $\mathcal{L}_{sdf}$ and $\mathcal{L}_{normal}$: Not given in the sources.
*   SDF network architecture: Based on NeuS [S1].

**Edge Cases and Pitfalls:**
*   The method relies on the quality of the underlying NeuS SDF initialization.
*   Precision issues may arise if the SDF network does not converge to a tight surface representation [S1].

## Subtopics

### GSDF
Encourages consistency between SDF and 3DGS to enhance geometry reconstruction quality [S1].

### Surface and Geometry Alignment
A broader category of methods including NeuSG, 3DGSR, and others that enforce geometric priors [S1].

## Operating characteristics
*   **Time Complexity:** Higher than vanilla 3DGS due to the additional SDF network queries and geometric loss computations [S1].
*   **Memory Complexity:** Increased by the storage and computation of the SDF MLP [S1].
*   **Strengths:** Produces cleaner, more coherent surfaces suitable for mesh extraction compared to vanilla 3DGS [S1][S2, unofficial].
*   **Limitations:** More complex training pipeline; requires careful balancing of geometric loss weights [S1].
*   **When to use:** When explicit surface geometry is required for downstream tasks like physics or DCC pipelines [S2, unofficial].
*   **When not to use:** When raw rendering speed is the primary concern and surface coherence is not critical [S2, unofficial].

## Background
NeuSG was introduced as a method to combine the surface reconstruction capabilities of NeuS with the rendering efficiency of 3DGS [S1]. It addresses the limitation of vanilla 3DGS, which produces floating ellipsoids rather than explicit surfaces [S2, unofficial]. The method builds on the NeuS paper [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
