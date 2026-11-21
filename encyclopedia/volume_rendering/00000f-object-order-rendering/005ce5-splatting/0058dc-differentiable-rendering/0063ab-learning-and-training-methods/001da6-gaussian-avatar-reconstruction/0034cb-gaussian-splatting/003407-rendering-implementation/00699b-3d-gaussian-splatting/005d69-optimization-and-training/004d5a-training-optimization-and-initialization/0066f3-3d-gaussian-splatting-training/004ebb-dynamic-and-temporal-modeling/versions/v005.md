# Dynamic and Temporal Modeling

Also called: Dynamic Scene Reconstruction, 4D Gaussian Splatting, Time-dependent NeRF

## Overview
Dynamic and Temporal Modeling extends static 3D Gaussian Splatting (3DGS) and Neural Radiance Fields (NeRF) to represent scenes that change over time, enabling novel view synthesis for non-rigid or dynamic content [S1, S2]. Unlike static methods that assume a fixed geometry, these approaches parameterize the evolution of scene attributes—such as Gaussian positions, rotations, opacities, or neural field densities—across a temporal dimension [S1, S2]. The key distinction from sibling methods lies in the representation of time: some methods use explicit parametric functions (e.g., Fourier series for Gaussians) to ensure temporal continuity [S2], while others use implicit deformation fields or canonical space mappings to warp dynamic points to a static reference [S1, S3].

## Shared principles
All methods in this family address the problem of representing a 4D spatiotemporal volume $D(x, y, z, t)$ using a set of input images captured at different times $t$. The core mechanism involves decomposing the scene into static and dynamic components or modeling the temporal evolution of attributes directly.

1.  **Temporal Parameterization:** Attributes that vary with time (position $\mathbf{c}$, rotation $\mathbf{q}$, opacity $\alpha$, density $\sigma$) are expressed as functions of time $t$. Common functional forms include:
    *   **Polynomial/Linear:** $\mathbf{q}(t) = \mathbf{q}_0 + \mathbf{m}t$ [S2].
    *   **Fourier Series:** $\mathbf{c}(t) = \mathbf{c}_0 + \sum_{k=0}^{K} (\mathbf{a}_k \cos(k \omega t) + \mathbf{b}_k \sin(k \omega t))$ [S2].
    *   **Neural Deformation:** A neural network $\Psi_t(x, t)$ predicts displacement $\Delta x$ to warp points to a canonical space [S1, S3].
    *   **Implicit Fields:** Time-dependent NeRFs map $(x, d, t)$ to density and color, often using local neighborhoods of frames [S4].

2.  **Canonical Space Mapping:** Many methods (e.g., D-NeRF, Nerfies) introduce a canonical configuration (usually at $t=0$) where the scene is static. A deformation network maps dynamic coordinates $x$ at time $t$ to canonical coordinates $x'$:
    $$ x' = x + \Psi_t(x, t) $$
    The radiance field is then queried in the canonical space: $[\sigma, c] = \Psi_x(x', d)$ [S1, S3].

3.  **Rendering with Dynamics:** The rendering process integrates the time-dependent attributes. For Gaussians, the position and rotation are computed for the specific time $t$ of the camera ray before rasterization [S2]. For NeRFs, the density $\sigma$ and color $c$ are queried at the deformed or time-conditioned point [S1, S4].

4.  **Loss Functions:** Optimization typically minimizes photometric loss (L1 or LPIPS) between rendered and ground truth images [S1, S2]. Additional constraints often include:
    *   **Temporal Smoothness:** Penalizing abrupt changes in attributes between adjacent frames [S1].
    *   **Rigid/Flow Constraints:** Using optical flow or KNN rigid loss to guide deformation [S1].
    *   **Regularization:** Elastic or background regularization to prevent overfitting in deformation fields [S5].

## Subtopics

### [Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md)
NeTF is a framework for Non-Line-of-Sight (NLOS) imaging that recovers a 5D transient function (spatial location and direction) implicitly using an MLP. It differs from standard NeRF by parametrizing data on spherical wavefronts rather than rays, using MCMC to handle sparse sampling.

### [SWAG](6875-swag.md)
SWAG extends 3DGS by training transient Gaussians to ignore occluders, separating static scene geometry from dynamic objects. It uses appearance embeddings and explicitly excludes transient Gaussians from rendering, unlike methods that rely on depth regularization.

### [Projected Color-Guided Dynamic NeRF](7523-projected-color-guided-dynamic-nerf.md)
OD-NeRF replaces explicit temporal deformation fields with conditioning on multi-view projected colors. It leverages the invariance of projected colors across frames to implicitly track correspondence, allowing on-the-fly training with minimal optimization iterations per frame.

### [On-the-Fly Dynamic NeRF Training](7582-on-the-fly-dynamic-nerf-training.md)
This method trains dynamic NeRFs concurrently with frame acquisition, using a first-order Markov assumption to maintain scalability. It minimizes rendering loss frame-by-frame, avoiding the need to optimize over full video sequences.

### [D-NeRF](8660-d-nerf.md)
D-NeRF synthesizes novel views of dynamic scenes by decomposing learning into a deformation network (mapping points to canonical space) and a canonical network (regressing density/color). It uses shared canonical anchors to exploit temporal redundancy, unlike naive $(x,y,z,t)$ mappings.

### [Dynamic Object Embedding in Static Scenes](8796-dynamic-object-embedding-in-static-scenes.md)
This method integrates dynamic objects into static NeRF scenes using a local deformation field and spatial alignment. It reduces parameter count by 95% compared to full 4D volume models by isolating motion to specific objects.

### [Dynamic 3D Gaussian Splatting (Katsumata et al.)](9168-dynamic-3d-gaussian-splatting-katsumata-et-al.md)
This method models dynamic scenes by parameterizing the time-dependent evolution of Gaussian positions (via Fourier series) and orientations (via linear functions) while keeping other attributes static. It ensures temporal continuity and avoids overfitting associated with large parameter counts.

### [DaReNeRF](13782-darenerf.md)
DaReNeRF addresses shift variance in dynamic NeRFs by using a direction-aware representation based on the Dual-Tree Complex Wavelet Transform (DTCWT). It decomposes the 4D volume into Hadamard products of 2D planes, reducing memory complexity and capturing directional features.

#### Temporal 3D Gaussian Splatting
- [S1] Introduces a temporal radial basis function for temporal opacity to model emerging/vanishing content, and polynomial functions for motion/rotation. Spherical harmonics are replaced by features (base color, view-related, time-related) processed by an MLP to generate residual color. New Gaussians are sampled at under-optimized positions based on training error and coarse depth.

#### SWAGS
- [S1] Divides dynamic sequences into windows based on motion amount, training separate dynamic 3DGS models with different canonical spaces and deformation fields. The deformation field uses a tunable MLP focused on dynamic parts. Fine-tuning ensures temporal consistency between windows using overlapping frames, fixing the MLP and optimizing only the canonical representation.

#### 3DGStream
- [S1] Allows online training of 3DGS for dynamic scene reconstruction by modeling frame transformations as a neural transformation cache and adaptively adding 3D Gaussians to handle emerging objects.

#### Blur Modeling in 3D GS
- [S2] Techniques to explicitly incorporate blur modeling (motion/defocus) during training. Steps include coarse-to-fine kernel optimization and photometric bundle adjustment.

#### Multi-scale Motion-Compensated Reconstruction
- [S3] A procedure coupled with DynART/P-DVC to handle large displacements. It resolves large-scale features of microstructure and motion first, then progressively enriches the description with finer details.

#### Time-dependent NeRF
- [S4] Represents scene geometry and appearance at each time step $t$ using neighboring frames. Appearance is mapped via $c = \phi_c(x, d)$ and geometry via $s = \phi_g(x)$. Density $\sigma_i$ is derived from SDF $s_i$ using $\sigma_i = \max((\Phi(s_i) - \Phi(s_{i+1}))/\Phi(s_i), 0)$ with $\Phi(s) = 1/(1+e^{1-\gamma s})$. Rendering uses $\hat{C}(r) = \sum L_i \sigma_i c_i$. SDF flows constrain continuous camera motion.

#### NeRF with Deformation Fields (Nerfies)
- [S5] Uses an additional MLP to map observation frame coordinates to deformed canonical coordinates. Adds elastic, background, and coarse-to-fine deformation regularization via adaptive masking of positional encoding to handle non-rigid transformations.

## Background
Dynamic scene reconstruction has evolved from explicit deformation fields in NeRFs to explicit parametric models in 3DGS. D-NeRF (2021) pioneered the canonical space approach for NeRFs [S1, S3]. Nerfies (2020) introduced deformation fields with regularization for non-rigid scenes [S5]. More recently, Dynamic 3D Gaussian Splatting (Katsumata et al.) adapted 3DGS for dynamic scenes using Fourier series for position and linear functions for rotation [S2]. Other approaches like SWAG and SWAGS focus on handling transient objects and windowed optimization [S1]. The field also includes specialized applications like NLOS imaging (NeTF) [S1] and on-the-fly training (OD-NeRF) [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S3] Dynamic Tomographic Reconstruction of Deforming Volumes, https://doi.org/10.3390/ma11081395, fetched 2026-10-26
[S4] Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video, https://doi.org/10.1109/cvpr52734.2025.01071, fetched 2026-10-26
[S5] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
