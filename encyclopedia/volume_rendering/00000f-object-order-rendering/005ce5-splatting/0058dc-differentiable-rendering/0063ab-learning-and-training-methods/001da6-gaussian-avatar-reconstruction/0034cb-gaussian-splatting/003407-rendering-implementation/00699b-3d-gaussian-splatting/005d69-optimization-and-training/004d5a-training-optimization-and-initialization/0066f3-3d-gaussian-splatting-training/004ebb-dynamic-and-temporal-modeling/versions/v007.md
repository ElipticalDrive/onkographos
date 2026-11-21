# Dynamic and Temporal Modeling

Also called: Dynamic Scene Reconstruction, 4D Modeling, Deformation Fields

## Overview
Dynamic and Temporal Modeling addresses the problem of reconstructing and rendering scenes that change over time, represented as a 4D spatiotemporal volume $D(x, y, z, t)$, from input images captured at different time instants $t$ [S1, S3]. It distinguishes itself from static reconstruction methods by explicitly modeling temporal evolution—either through neural deformation fields that warp coordinates to a canonical space, explicit attribute parameterization (e.g., time-dependent Gaussians), or frequency-domain representations—rather than treating motion as noise or requiring full offline video sequences for all methods [S1, S3]. This family enables novel view synthesis at arbitrary time instants, on-the-fly streaming, and handling of complex non-rigid motions [S1, S3].

## Shared principles
The core mechanism for dynamic modeling involves representing the 4D volume $D(x, y, z, t)$ using inputs captured at various times $t$. The shared principles across the subtopics are:

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
**What it is:** An on-the-fly training method (OD-NeRF) that replaces temporal deformation fields with a NeRF model conditioned on multi-view projected colors. It leverages the relative invariance of projected colors to implicitly track point correspondence between consecutive frames.
**Distinction:** Unlike sibling methods that rely on explicit temporal parameterization or deformation networks which extrapolate poorly to unseen times, OD-NeRF enables rapid convergence with minimal optimization iterations per frame by using a hash voxel grid and rejection sampling guided by an occupancy grid.
**When to choose:** When streaming dynamic scenes frame-by-frame is required, and the scene contains static or slowly moving elements where projected color invariance holds.
**Link:** [7523-projected-color-guided-dynamic-nerf.md](7523-projected-color-guided-dynamic-nerf.md)

### [On-the-Fly Dynamic NeRF Training](7582-on-the-fly-dynamic-nerf-training.md)
**What it is:** A method for training a dynamic Neural Radiance Field concurrently with the acquisition of training frames, enabling the streaming of a dynamic scene while it unfolds. It estimates the radiance field for the current time step $t_k$ using only the images up to $C_{0:k}$ and the radiance field from the previous step $F_{0:k-1}$.
**Distinction:** It is distinguished by its use of a first-order Markov assumption to limit computational complexity, allowing for frame-by-frame optimization with minimal iterations, unlike standard dynamic NeRFs that require full video sequences for offline 4D reconstruction.
**When to choose:** For real-time or near-real-time reconstruction of forward-facing dynamic scenes where computational scalability is critical.
**Link:** [7582-on-the-fly-dynamic-nerf-training.md](7582-on-the-fly-dynamic-nerf-training.md)

### [D-NeRF](8660-d-nerf.md)
**What it is:** A method that synthesizes novel views at arbitrary time instants for dynamic scenes with complex non-rigid geometries, optimizing a deformable volumetric function from a sparse set of monocular input images.
**Distinction:** It distinguishes itself from naive time-extended NeRFs by decomposing the representation into a canonical static radiance field and a learned deformation field, effectively exploiting temporal redundancy to handle rigid, articulated, and non-rigid motions.
**When to choose:** When training from sparse monocular inputs (one view per time) and the scene exhibits complex non-rigid motion that can be mapped to a canonical space.
**Link:** [8660-d-nerf.md](8660-d-nerf.md)

### [DaReNeRF](13782-darenerf.md)
**What it is:** A differentiable volume rendering method for novel view synthesis of dynamic scenes that replaces standard plane-based decompositions with a direction-aware representation based on the Dual-Tree Complex Wavelet Transform (DTCWT).
**Distinction:** Unlike sibling methods that rely on neural deformation fields to warp coordinates to a canonical space, DaReNeRF directly models the 4D spatiotemporal volume using learned complex wavelet coefficients, offering a more robust frequency-domain representation of motion that solves shift variance and direction selectivity issues.
**When to choose:** When dealing with dynamic scenes where motion causes ghosting artifacts in standard wavelet or plane-based NeRFs, and a frequency-domain approach is preferred over deformation networks.
**Link:** [13782-darenerf.md](13782-darenerf.md)

### [Blur and Transient Modeling](13851-blur-and-transient-modeling.md)
**What it is:** A family of methods that addresses the degradation of reconstruction quality in dynamic scenes and real-world captures caused by temporal inconsistencies (transients) and optical blurs (motion/defocus).
**Distinction:** It distinguishes itself from static reconstruction methods by explicitly modeling non-static elements as separate entities or incorporating blur kernels into the rendering pipeline, rather than treating them as noise or artifacts.
**When to choose:** When the input data contains significant motion blur, defocus blur, or transient artifacts (e.g., moving people, reflections) that degrade standard dynamic reconstruction.
**Link:** [13851-blur-and-transient-modeling.md](13851-blur-and-transient-modeling.md)

### [Deformation Field Modeling](13852-deformation-field-modeling.md)
**What it is:** A method addressing the problem of representing a 4D spatiotemporal volume $D(x, y, z, t)$ by mapping dynamic coordinates to a static canonical space using a neural network to predict displacement vectors.
**Distinction:** It differs from explicit attribute parameterization (e.g., Dynamic 3D Gaussian Splatting) by using a neural network to predict displacement vectors, allowing for complex, non-rigid, and non-periodic motion that explicit functions cannot easily capture.
**When to choose:** When the scene motion is complex, non-rigid, and non-periodic, requiring a flexible neural representation rather than fixed basis functions.
**Link:** [13852-deformation-field-modeling.md](13852-deformation-field-modeling.md)

### Temporal 3D Gaussian Splatting
**Definition:** A method that introduces a temporal radial basis function to represent temporal opacity, modeling emerging or vanishing content. Polynomial functions model motion and rotation. Spherical harmonics are replaced with features consisting of base color, view-related feature, and time-related feature. The latter two are translated into residual color through an MLP added to the base color [S1].
**Steps:** During optimization, new 3D Gaussians are sampled at under-optimized positions based on training error and coarse depth [S1].
**Source:** [S1]

### SWAGS
**Definition:** A method that divides the dynamic sequence into different windows based on motion amount, training separate dynamic 3DGS models in different windows with different canonical spaces and deformation fields. The deformation field uses a tunable MLP that focuses more on modeling the dynamic part of the scene [S1].
**Steps:** Fine-tuning ensures temporal consistency between windows using overlapping frames to add constraints. The MLP is fixed and only the canonical representation is optimized during fine-tuning [S1].
**Source:** [S1]

### 3DGStream
**Definition:** A method that allows online training of 3DGS into dynamic scene reconstruction by modeling the transformation between frames as a neural transformation cache and adaptively adding 3D Gaussians to handle emerging objects [S1].
**Source:** [S1]

### Multi-scale Motion-Compensated Reconstruction
**Definition:** A procedure coupled with the two-step DynART/P-DVC algorithm to handle large displacement amplitudes by resolving large-scale features first and progressively enriching with finer details [S2].
**Steps:** Resolve large-scale features of both microstructure and motion. Progressively enrich the description with finer details [S2].
**Source:** [S2]

### Time-dependent NeRF
**Definition:** A NeRF variant that represents scene geometry and appearance at each specific time step $t$, trained using neighboring frames around time $t$, to capture local scene geometry and motion [S3].
**Steps:** Represent scene appearance using mapping function $c = \phi_c(x, d)$ taking point $x \in \mathbb{R}^3$ and viewing direction $d \in \mathbb{R}^2$ to predict color $c \in \mathbb{R}^3$. Represent scene geometry via mapping function $s = \phi_g(x)$ which maps a 3D point $x$ to a signed distance value $s \in \mathbb{R}$. For a camera ray $r(h) = o + hd$, sample $K$ 3D points $\{x_i(r) = r(h_i) \mid i = 1, \dots, K, h_i < h_{i+1}\}$. Use the time-dependent NeRF to compute SDF flows, which are used to constrain the learning of continuous camera motion via the linear relationship between SDF flow and scene flow [S3].
**Formulas:** Convert signed distance $s_i$ to density $\sigma_i$ using: $\sigma_i = \max((\Phi(s_i) - \Phi(s_{i+1}))/\Phi(s_i), 0)$, where $\Phi(s) = 1/(1+e^{1-\gamma s})$, $\gamma$ is a learnable parameter, and $i$ is the point index. Render pixel color $\hat{C}(r)$ via volume rendering: $\hat{C}(r) = \sum_{i=1}^K L_i \sigma_i c_i$, where $L_i = \prod_{j=1}^{i-1} (1 - \sigma_j)$ is the accumulated transmittance [S3].
**Source:** [S3]

## Background
Dynamic scene reconstruction has evolved from offline 4D reconstruction methods to real-time and on-the-fly approaches. Early methods like D-NeRF [S1, S3] established the canonical space + deformation field paradigm for handling non-rigid motion from sparse inputs. More recent advancements, such as OD-NeRF [S1], focus on streaming and on-the-fly training using Markov assumptions to reduce computational load. In the context of 3D Gaussian Splatting, methods like Temporal 3D Gaussian Splatting [S1], SWAGS [S1], and 3DGStream [S1] have adapted the explicit representation to handle dynamics through temporal basis functions, windowed training, and neural transformation caches. Frequency-domain approaches like DaReNeRF [S1] offer alternatives to deformation fields. The field also addresses practical issues like blur and transients [S1] and integrates with other domains like tomography [S2] and joint camera motion estimation [S3].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Dynamic Tomographic Reconstruction of Deforming Volumes, https://doi.org/10.3390/ma11081395, fetched 2026-10-26
[S3] Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video, https://doi.org/10.1109/cvpr52734.2025.01071, fetched 2026-10-26
