# Dynamic and Temporal Modeling

Also called: Dynamic Scene Reconstruction, 4D Radiance Fields, Deformation Field Modeling

## Overview
Dynamic and Temporal Modeling addresses the reconstruction and rendering of scenes that change over time, extending static radiance field representations (such as NeRF and 3D Gaussian Splatting) to handle non-rigid motion, camera movement, and transient phenomena. It solves the problem of synthesizing novel views at arbitrary time steps from sparse or streaming input sequences where the scene geometry and appearance are not fixed. This family is distinguished from static methods by its explicit or implicit parameterization of time $t$ as a core dimension of the scene representation, often employing canonical space mappings or temporal basis functions to decouple geometry from motion. It differs from sibling methods like [Physical Parameter Optimization](4111-physical-parameter-optimization.md) by focusing on spatiotemporal continuity rather than physical light transport properties, and from [WildGaussians](6064-wildgaussians.md) by providing the foundational temporal modeling mechanisms that such specialized variants build upon.

## Shared principles
The shared principles across all dynamic and temporal modeling methods in this family are:

1.  **Temporal Parameterization:** Attributes that vary with time, such as position $\mathbf{c}$, rotation $\mathbf{q}$, opacity $\alpha$, density $\sigma$, or color $\mathbf{c}$, are expressed as functions of time $t$ [S1, S2, S3]. Common functional forms include:
    *   **Polynomial/Linear:** $\mathbf{q}(t) = \mathbf{q}_0 + \mathbf{m}t$ [S2].
    *   **Fourier Series:** $\mathbf{c}(t) = \mathbf{c}_0 + \sum_{k=0}^{K} (\mathbf{a}_k \cos(k \omega t) + \mathbf{b}_k \sin(k \omega t))$ [S2].
    *   **Neural Deformation:** A neural network $\Psi_t(x, t)$ predicts a displacement $\Delta x$ to warp dynamic coordinates to a canonical space [S1, S3].
    *   **Implicit Fields:** Time-dependent NeRFs map $(x, d, t)$ to density and color, often using local neighborhoods of frames [S4].

2.  **Canonical Space Mapping:** Many methods (e.g., D-NeRF, Nerfies) introduce a canonical configuration, typically at $t=0$, where the scene is static. A deformation network maps dynamic coordinates $x$ at time $t$ to canonical coordinates $x'$:
    $$ x' = x + \Psi_t(x, t) $$
    The radiance field is then queried in the canonical space: $[\sigma, c] = \Psi_x(x', d)$ [S1, S3].

3.  **Rendering with Dynamics:** The rendering process integrates time-dependent attributes. For Gaussian-based methods, position and rotation are computed for the specific time $t$ of the camera ray before rasterization [S2]. For NeRF-based methods, density $\sigma$ and color $c$ are queried at the deformed or time-conditioned point [S1, S4].

4.  **Loss Functions:** Optimization typically minimizes photometric loss (L1 or LPIPS) between rendered and ground truth images [S1, S2]. Additional constraints often include:
    *   **Temporal Smoothness:** Penalizing abrupt changes in attributes between adjacent frames [S1].
    *   **Rigid/Flow Constraints:** Using optical flow or KNN rigid loss to guide deformation [S1].
    *   **Regularization:** Elastic or background regularization to prevent overfitting in deformation fields [S5].

## Subtopics

### [Projected Color-Guided Dynamic NeRF](7523-projected-color-guided-dynamic-nerf.md)
Projected Color-Guided Dynamic NeRF (OD-NeRF) is an on-the-fly training method for dynamic scenes that replaces temporal deformation fields with a NeRF model conditioned on multi-view projected colors. By leveraging the observation that the projected color of a corresponding 3D point remains invariant across consecutive frames, the model implicitly tracks point correspondence, allowing it to effectively utilize radiance fields from previous frames to optimize the current frame. Unlike sibling methods such as [D-NeRF](8660-d-nerf.md) or [DaReNeRF](13782-darenerf.md) which rely on explicit temporal inputs or deformation networks that extrapolate poorly to unseen times, OD-NeRF requires the minimum number of optimization iterations per frame and exhibits superior temporal extrapolation capabilities.

### [On-the-Fly Dynamic NeRF Training](7582-on-the-fly-dynamic-nerf-training.md)
On-the-fly dynamic NeRF training is a method for reconstructing dynamic scenes by training a neural radiance field concurrently with the acquisition of training frames, enabling the streaming of the scene as the event unfolds. Unlike standard dynamic NeRFs that require full video sequences for offline 4D reconstruction, this approach estimates the radiance field for the current time step $t_k$ using only the images up to $t_k$ and the radiance field from the previous step $t_{k-1}$. It distinguishes itself by applying a first-order Markov assumption to limit the dependency history, allowing for minimal optimization iterations per frame and interactive-speed streaming.

### [D-NeRF](8660-d-nerf.md)
D-NeRF is a method for synthesizing novel views at an arbitrary point in time of dynamic scenes with complex non-rigid geometries, optimizing an underlying deformable volumetric function from a sparse set of input monocular views without ground-truth geometry or multi-view images. It sets itself apart from naive extensions of NeRF (which map $(x, y, z, t)$ directly to density and radiance) by decomposing the learning process into two modules: a deformation network that maps dynamic coordinates to a static canonical space, and a canonical radiance network that predicts density and color from that space. This decomposition effectively exploits temporal redundancy, allowing the model to handle rigid, articulated, and non-rigid motions from a single camera moving around the scene.

### [DaReNeRF](13782-darenerf.md)
DaReNeRF is a method for novel view synthesis of dynamic scenes that addresses the shift variance and direction ambiguity inherent in traditional Discrete Wavelet Transforms (DWT) by employing a direction-aware representation. It decomposes the 4D spatiotemporal volume into a sum of Hadamard products of 2D plane-based representations and learned vectors, utilizing an inverse dual-tree complex wavelet transformation (DTCWT) to recover plane information from six distinct orientations. Unlike baseline plane-based decompositions that suffer from checkerboard artifacts and poor motion handling, DaReNeRF ensures shift invariance and directional selectivity, reducing memory complexity from $O(N^3 T F)$ to $O(R N^2 T F)$ while maintaining high-fidelity rendering of complex motions.

### [Blur and Transient Modeling](13851-blur-and-transient-modeling.md)
This family addresses temporal dynamics and optical artifacts in 3D Gaussian Splatting (3DGS) and related radiance field representations. It solves the problem of reconstructing scenes with moving objects, transient occluders, and optical blurs (motion/defocus) that standard static 3DGS fails to capture. It distinguishes itself from sibling methods like [D-NeRF](8660-d-nerf.md) by operating directly in the Gaussian primitive space (e.g., SWAG) or using specialized neural transient fields for non-line-of-sight (NLOS) imaging (NeTF), rather than relying solely on continuous deformation fields for general dynamic scenes.

### [Deformation Field Modeling](13852-deformation-field-modeling.md)
Deformation Field Modeling represents dynamic scenes by mapping time-dependent 3D coordinates to a static canonical space, allowing standard static radiance field queries to render temporal variations. It differs from explicit attribute parameterization (e.g., Fourier-based Gaussian motion) by learning a continuous displacement function $\Psi_t$, which handles complex non-rigid deformations more flexibly than fixed analytical functions.

### Temporal 3D Gaussian Splatting
- [S1] A method that introduces a temporal radial basis function to represent temporal opacity, modeling emerging or vanishing content. Polynomial functions model motion and rotation.
- [S1] Spherical harmonics are replaced with features consisting of base color, view-related feature, and time-related feature. The latter two are translated into residual color through an MLP added to the base color.
- [S1] During optimization, new 3D Gaussians are sampled at under-optimized positions based on training error and coarse depth.

### SWAGS
- [S1] A method that divides the dynamic sequence into different windows based on motion amount, training separate dynamic 3DGS models in different windows with different canonical spaces and deformation fields.
- [S1] The deformation field uses a tunable MLP that focuses more on modeling the dynamic part of the scene.
- [S1] Fine-tuning ensures temporal consistency between windows using overlapping frames to add constraints. The MLP is fixed and only the canonical representation is optimized during fine-tuning.

### 3DGStream
- [S1] A method that allows online training of 3DGS into dynamic scene reconstruction by modeling the transformation between frames as a neural transformation cache and adaptively adding 3D Gaussians to handle emerging objects.

### Multi-scale Motion-Compensated Reconstruction
- [S2] A procedure coupled with the two-step DynART/P-DVC algorithm to handle large displacement amplitudes by resolving large-scale features first and progressively enriching with finer details.
- [S2] Resolve large-scale features of both microstructure and motion.
- [S2] Progressively enrich the description with finer details.

### Time-dependent NeRF
- [S3] A NeRF variant that represents scene geometry and appearance at each specific time step t, trained using neighboring frames around time t, to capture local scene geometry and motion.
- [S3] Represent scene appearance using mapping function $c = \phi_c(x, d)$ taking point $x \in \mathbb{R}^3$ and viewing direction $d \in \mathbb{R}^2$ to predict color $c \in \mathbb{R}^3$.
- [S3] Represent scene geometry via mapping function $s = \phi_g(x)$ which maps a 3D point $x$ to a signed distance value $s \in \mathbb{R}$.
- [S3] For a camera ray $r(h) = o + hd$, sample $K$ 3D points $\{x_i(r) = r(h_i) \mid i = 1, \dots, K, h_i < h_{i+1}\}$.
- [S3] Use the time-dependent NeRF to compute SDF flows, which are used to constrain the learning of continuous camera motion via the linear relationship between SDF flow and scene flow.
- [S3] Convert signed distance $s_i$ to density $\sigma_i$ using: $\sigma_i = \max\left(\frac{\Phi(s_i) - \Phi(s_{i+1})}{\Phi(s_i)}, 0\right)$, where $\Phi(s) = \frac{1}{1+e^{1-\gamma s}}$, $\gamma$ is a learnable parameter, and $i$ is the point index.
- [S3] Render pixel color $\hat{C}(r)$ via volume rendering: $\hat{C}(r) = \sum_{i=1}^{K} L_i \sigma_i c_i$, where $L_i = \prod_{j=1}^{i-1} (1 - \sigma_j)$ is the accumulated transmittance.

## Background
Dynamic scene reconstruction has evolved from explicit multi-view stereo techniques to implicit neural representations. D-NeRF [S1, S3] was a seminal work that introduced the canonical space deformation approach for dynamic NeRFs, enabling novel view synthesis from monocular inputs. Subsequent works like DaReNeRF [S1] improved upon plane-based decompositions to handle shift variance. In the context of 3D Gaussian Splatting, methods like Temporal 3DGS [S1] and SWAGS [S1] adapted the explicit primitive-based representation to handle temporal dynamics, while 3DGStream [S1] enabled online reconstruction. The field also intersects with tomographic reconstruction, where multi-scale motion-compensated algorithms [S2] handle large displacements. More recently, time-dependent NeRFs [S3] have been used to jointly optimize camera motion and scene geometry from monocular video by leveraging local temporal neighborhoods.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Dynamic Tomographic Reconstruction of Deforming Volumes, https://doi.org/10.3390/ma11081395, fetched 2026-10-26
[S3] Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video, https://doi.org/10.1109/cvpr52734.2025.01071, fetched 2026-10-26
