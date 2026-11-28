# 4D-Rotor Gaussian Splatting

Also called: 4D-Rotor GS, Clifford Algebra 4DGS

## Overview
4D-Rotor Gaussian Splatting (4DRotorGS) is a differentiable rendering method for dynamic scenes that lifts 3D Gaussians into 4D space $(x, y, z, t)$ using a rotor-based rotation representation from Clifford algebra [S3, S4, unofficial]. Unlike sibling methods such as [Real-time 4DGS](7335-real-time-4dgs.md) which use dual-quaternions that entangle spatial and temporal rotations, 4DRotorGS provides a spatial-temporal separable and interpretable rotation representation [S3, unofficial]. It avoids the numerical instability of directly inverting the 3D covariance matrix by leveraging block matrix inversion formulas on the 4D covariance to compute sliced 3D parameters [S3, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $N$ 4D Gaussians, each defined by a 4D center $\boldsymbol{\mu}_{4D}$, a 4D covariance $\boldsymbol{\Sigma}_{4D}$, spherical harmonics coefficients (or RGB color), and opacity $o$.
*   Camera parameters (extrinsics $\mathbf{V}$, intrinsics) and a specific timestamp $t$ for the frame to render.
*   Scene scales for normalizing 4D neighbor distances [S3, unofficial].

**Outputs:**
*   A 2D rendered image $C$ and depth/alpha buffers.

### Data Structures
**4D Rotor $\mathbf{r}$:**
A 4D rotor consists of 8 components stored as a vector of size 8: $[s, b_{01}, b_{02}, b_{12}, b_{03}, b_{13}, b_{23}, p]$ [S3, unofficial]. These correspond to the Clifford algebra basis elements $1, \mathbf{e}_{01}, \mathbf{e}_{02}, \mathbf{e}_{12}, \mathbf{e}_{03}, \mathbf{e}_{13}, \mathbf{e}_{23}, \mathbf{e}_{0123}$, where $\mathbf{e}_{ij} = \mathbf{e}_i \wedge \mathbf{e}_j$ and $\mathbf{e}_{0123} = \mathbf{e}_0 \wedge \mathbf{e}_1 \wedge \mathbf{e}_2 \wedge \mathbf{e}_3$ [S3, unofficial]. The axes $0,1,2,3$ correspond to $x,y,z,t$ [S3, unofficial].

**4D Covariance Block Matrices:**
The 4D covariance $\boldsymbol{\Sigma}_{4D}$ and its inverse $\boldsymbol{\Sigma}_{4D}^{-1}$ are partitioned into symmetric block matrices [S3, unofficial]:
$$
\boldsymbol{\Sigma}_{4D} = \begin{pmatrix} \mathbf{U} & \mathbf{V} \\ \mathbf{V}^T & \mathbf{W} \end{pmatrix}, \quad \boldsymbol{\Sigma}_{4D}^{-1} = \begin{pmatrix} \mathbf{A} & \mathbf{M} \\ \mathbf{M}^T & \mathbf{Z} \end{pmatrix}
$$
where $\mathbf{U}, \mathbf{A}$ are $3 \times 3$ matrices, $\mathbf{V}$ is $3 \times 1$, and $\mathbf{W}, \mathbf{Z}$ are scalars [S3, unofficial].

### Initialization
1.  **Spatial Position:** Randomly sample positions within an axis-aligned bounding box (e.g., $[-1.3, 1.3]^3$ for D-NeRF, or SfM range for Plenoptic Video) [S3, unofficial].
2.  **Time Mean:** Uniformly sample $\mu_t$ from $[0, 1]$ for D-NeRF or $[0, 10]$ for Plenoptic Video [S3, unofficial].
3.  **Time Scale:** Initialize $s_t$ to 0.1414 for D-NeRF or 1.414 for Plenoptic Video [S3, unofficial].
4.  **3D Scales:** Initialize $s_x, s_y, s_z$ [S3, unofficial].
5.  **Rotors:** Initialize the 8 components of the rotor $\mathbf{r}$ such that they satisfy the normalization conditions (see Normalization below) [S3, unofficial].

### Normalization of 4D Rotor
To ensure $\mathbf{r}$ represents a valid rotation, two conditions must be met [S3, unofficial]:
1.  $ps - b_{01}b_{23} + b_{02}b_{13} - b_{03}b_{12} = 0$
2.  $b_{01}^2 + b_{02}^2 + b_{03}^2 + b_{12}^2 + b_{13}^2 + b_{23}^2 + p^2 + s^2 = 1$

The normalization function $\mathcal{F}_{norm}$ corrects deviations using a gradient-based step [S3, unofficial]:
1.  Calculate squared norm $l^2 = \sum \text{components}^2$.
2.  Calculate $\Delta = b_{01}b_{23} - b_{02}b_{13} + b_{03}b_{12}$.
3.  Calculate $\varepsilon = ps - \Delta$.
4.  Calculate gradient $\nabla \varepsilon$:
    *   $\frac{\partial \varepsilon}{\partial p} = s, \quad \frac{\partial \varepsilon}{\partial s} = p$
    *   $\frac{\partial \varepsilon}{\partial b_{01}} = -b_{23}, \quad \frac{\partial \varepsilon}{\partial b_{23}} = -b_{01}$
    *   $\frac{\partial \varepsilon}{\partial b_{02}} = b_{13}, \quad \frac{\partial \varepsilon}{\partial b_{13}} = b_{02}$
    *   $\frac{\partial \varepsilon}{\partial b_{03}} = -b_{12}, \quad \frac{\partial \varepsilon}{\partial b_{12}} = -b_{03}$
5.  Calculate step size $\delta = \frac{-l^2 + \sqrt{l^4 - 4\varepsilon^2}}{2\varepsilon}$ [S3, unofficial].
6.  Update components: $\mathbf{r} \leftarrow \mathbf{r} + \delta \nabla \varepsilon$ [S3, unofficial].

### Rotor-to-Matrix Mapping
The 4D rotation matrix $\mathbf{R}_{4D}$ is constructed from the normalized rotor components [S1, S3, unofficial].
1.  Unpack normalized rotor $\mathbf{r}$ into $s, b_{01}, b_{02}, b_{12}, b_{03}, b_{13}, b_{23}, p$.
2.  Construct matrix $\mathbf{M}_l$ from components $s, b_{01}, b_{02}, b_{03}$ (interpreted as quaternion-like vector $\mathbf{q}_l$) using the pattern:
    $$
    \mathbf{M}_l = \begin{pmatrix}
    s & -b_{01} & -b_{02} & -b_{03} \\
    b_{01} & s & -b_{03} & b_{02} \\
    b_{02} & b_{03} & s & -b_{01} \\
    b_{03} & -b_{02} & b_{01} & s
    \end{pmatrix}
    $$
    *(Note: Source [S1, unofficial] describes a permutation and reshaping of `[a,-b,-c,-d, b, a,-d, c, c, d, a,-b, d,-c, b, a]`. The explicit matrix above is the standard left-multiplication matrix for a quaternion/vector in 4D Clifford algebra context derived from the basis order $1, \mathbf{e}_{01}, \mathbf{e}_{02}, \mathbf{e}_{03}$)*.
3.  Construct matrix $\mathbf{M}_r$ from components $b_{12}, b_{13}, b_{23}, p$ (interpreted as quaternion-like vector $\mathbf{q}_r$) using the pattern:
    $$
    \mathbf{M}_r = \begin{pmatrix}
    p & b_{12} & b_{13} & b_{23} \\
    -b_{12} & p & -b_{23} & b_{13} \\
    -b_{13} & b_{23} & p & -b_{12} \\
    -b_{23} & -b_{13} & b_{12} & p
    \end{pmatrix}
    $$
    *(Note: Source [S1, unofficial] describes a permutation and reshaping of `[p, q, r, s, -q, p,-s, r, -r, s, p,-q, -s,-r, q, p]`. The explicit matrix above corresponds to the right-multiplication structure for the remaining basis elements)*.
4.  Compute $\mathbf{R}_{4D} = \mathbf{M}_l \mathbf{M}_r$.
5.  Flip the last two dimensions of $\mathbf{R}_{4D}$ to account for coordinate system conventions [S1, unofficial].

### Temporal Slicing
At a given timestamp $t$, the 4D Gaussian is sliced into a 3D Gaussian [S3, unofficial].
1.  Extract blocks $\mathbf{U}, \mathbf{V}, \mathbf{W}$ from $\boldsymbol{\Sigma}_{4D}$.
2.  Compute the temporal decay parameter $\lambda = \mathbf{W}^{-1}$ [S3, unofficial].
3.  Compute the sliced 3D covariance $\boldsymbol{\Sigma}_{3D}$ without inverting $\mathbf{A}$ directly [S3, unofficial]:
    $$
    \boldsymbol{\Sigma}_{3D} = \mathbf{U} - \frac{\mathbf{V}\mathbf{V}^T}{\mathbf{W}}
    $$
4.  Compute the time-dependent 3D center $\boldsymbol{\mu}(t)$ [S3, unofficial]:
    $$
    \boldsymbol{\mu}(t) = (\mu_x, \mu_y, \mu_z)^T + (t - \mu_t) \frac{\mathbf{V}}{\mathbf{W}}
    $$
5.  The sliced 3D Gaussian density is:
    $$
    G_{3D}(\mathbf{x}, t) = e^{-\frac{1}{2}\lambda(t-\mu_t)^2} e^{-\frac{1}{2}[\mathbf{x}-\boldsymbol{\mu}(t)]^T \boldsymbol{\Sigma}_{3D}^{-1}[\mathbf{x}-\boldsymbol{\mu}(t)]}
    $$
    The term $e^{-\frac{1}{2}\lambda(t-\mu_t)^2}$ acts as a temporal envelope, causing the Gaussian to appear, peak at $t=\mu_t$, and vanish as $t$ moves away [S3, unofficial].

### Rendering
1.  Project the sliced 3D Gaussians to the 2D image plane using the standard 3DGS projection $\boldsymbol{\Sigma}' = \mathbf{J}\mathbf{V}\mathbf{V}^T\mathbf{J}^T$ [S3, unofficial].
2.  Sort Gaussians by depth.
3.  Perform alpha blending to compute pixel colors [S3, unofficial]:
    $$
    C = \sum_{i=1}^N c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$

### Optimization
The method uses Adam optimizer with specific learning rates [S3, unofficial]:
*   Positions/Times: $1.6 \times 10^{-4}$ (exponential decay to $1.6 \times 10^{-6}$ at step 30,000).
*   3D Scales/Time Scales: $5 \times 10^{-3}$.
*   Rotation: $1 \times 10^{-3}$.
*   SH Low Degree: $2.5 \times 10^{-3}$.
*   SH High Degrees: $1.25 \times 10^{-4}$.
*   Opacity: $0.05$.

**Loss Function:**
$$
L = (1 - \lambda_1)L_1 + \lambda_1 L_{ssim} + \lambda_2 L_{\text{entropy}} + \lambda_3 L_{\text{consistent4D}}
$$
1.  **L1 and SSIM:** Standard image reconstruction losses [S3, unofficial].
2.  **Entropy Loss:** Encourages opacity $o_i$ to approach 0 or 1, reducing floaters [S3, unofficial]:
    $$
    L_{\text{entropy}} = \frac{1}{N} \sum_{i=1}^N -o_i \log(o_i)
    $$
3.  **4D Consistency Loss:** Enforces similar motion among 4D neighbors [S3, unofficial]:
    $$
    L_{\text{consistent4D}} = \frac{1}{N} \sum_{i=1}^N \left\| \mathbf{s}_i - \frac{1}{K} \sum_{j \in \Omega_i} \mathbf{s}_j \right\|_1
    $$
    where $\mathbf{s}_i$ is the speed term derived from slicing ($\mathbf{V}/\mathbf{W}$), $K$ is the number of neighbors, and $\Omega_i$ is the set of $K$ nearest 4D neighbors. Neighbors are found using Euclidean distance in 4D, balanced by scene scales [S3, unofficial].

**Training Details:**
*   Densification every 100 steps (steps 500–15,000) [S3, unofficial].
*   Opacity reset every 3,000 steps [S3, unofficial].
*   Pruning of near-zero opacity Gaussians [S3, unofficial].

## Subtopics

### 4D Scaling and Rotation Construction
Constructs the 4D transformation matrix by combining scaling and rotation. Initializes a zero matrix $\mathbf{L}$ of shape $(N, 4, 4)$, sets diagonal elements to scaling factors $s_x, s_y, s_z, s_t$, computes the rotation matrix $\mathbf{R}$ using the rotor normalization and mapping steps, and returns $\mathbf{R} \mathbf{L}$ [S1, unofficial].

### 4D Rotor Normalization
The specific numerical procedure to enforce the Clifford algebra constraints on the 8-component rotor vector. It calculates the squared norm $l^2$ and the constraint violation $\varepsilon$, then applies a gradient-based correction step $\delta$ to satisfy both the unit norm and the pseudo-scalar constraint $ps - \Delta = 0$ [S3, unofficial].

## Operating characteristics
*   **Complexity:** Rendering complexity is similar to 3DGS, $O(N)$ per pixel for blending, with the overhead of computing the sliced 3D covariance and center at each timestamp [S3, unofficial].
*   **Performance:** Achieves real-time rendering speeds comparable to 3DGS when implemented in CUDA [S3, unofficial].
*   **Strengths:** Interpretable separation of spatial and temporal rotations; handles dynamic scenes with appearing/disappearing objects better than deformation-based methods; avoids numerical instability in covariance inversion [S3, unofficial].
*   **Limitations:** Entropy loss does not improve PSNR for dense-view reconstruction (e.g., vanilla 3DGS garden scene) but helps in sparse-view [S3, unofficial]. Requires careful initialization of time scales [S3, unofficial].
*   **Metrics:** On D-NeRF dataset, full model PSNR is 33.06. Removing KNN reduces it to 31.91; removing Entropy to 32.11; removing Batch to 32.64 [S3, unofficial].

## Background
4D-Rotor Gaussian Splatting was introduced by Li et al. in 2024 [S3, unofficial]. It builds on 3D Gaussian Splatting [Kerbl et al. 2023] by lifting Gaussians into 4D space. It contrasts with RealTime4DGS [Yang et al. 2024] by using Clifford algebra rotors instead of dual-quaternions for better interpretability and spatial-temporal separability [S3, unofficial]. The method is used for dynamic scene reconstruction from multi-view videos, such as the D-NeRF and Plenoptic Video datasets [S3, unofficial].

## Sources
[S1] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] 4D-Rotor Gaussian Splatting, https://arxiv.org/abs/2402.03307, fetched 2026-10-27 (unofficial)
[S4] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
