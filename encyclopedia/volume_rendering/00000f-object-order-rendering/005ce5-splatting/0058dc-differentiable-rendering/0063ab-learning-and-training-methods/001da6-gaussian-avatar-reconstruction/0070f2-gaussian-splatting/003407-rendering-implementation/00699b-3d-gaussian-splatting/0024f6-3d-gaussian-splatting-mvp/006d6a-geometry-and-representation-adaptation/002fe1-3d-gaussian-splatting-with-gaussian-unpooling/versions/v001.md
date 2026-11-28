# 3D Gaussian Splatting with Gaussian Unpooling

Also called: FSGS

## Overview
FSGS is a method for reconstructing 3D scenes from sparse-view inputs, addressing the challenge of insufficient initialization data for standard 3D Gaussian Splatting. It distinguishes itself from dense-view baselines by using Gaussian unpooling to expand a sparse set of Structure-from-Motion (SfM) points into a denser set of Gaussians, enabling faithful geometry reconstruction where standard methods fail due to lack of initial coverage [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A sparse set of input images with known camera poses, initial 3D point cloud from Structure-from-Motion (SfM), and a pre-trained 2D depth estimation network.
- **Outputs:** A set of 3D Gaussians (position, covariance, opacity, spherical harmonics coefficients) optimized to render the input views and novel views.

**Data Structures:**
The method operates on the standard 3D Gaussian Splatting data structure: a list of Gaussians, each defined by a mean position $\mathbf{\mu}$, a rotation quaternion $\mathbf{q}$, a scale vector $\mathbf{s}$, an opacity $\alpha$, and spherical harmonics coefficients $\mathbf{c}$. The initial set is sparse, derived from SfM.

**Steps:**
1. **Sparse Initialization:** Initialize the Gaussian set with points from the SfM reconstruction. These points serve as the "source" Gaussians [S1].
2. **Gaussian Unpooling:** Expand the sparse set of Gaussians into a denser set. This process, termed "Gaussian unpooling," identifies and generates new Gaussians from the existing sparse ones to fill gaps in the scene representation. The exact algorithmic details of the unpooling expansion (e.g., how many new Gaussians are created per source, their initial parameters) are not specified in the provided sources [S1][S2, unofficial].
3. **Depth Supervision:** Render depth images from the current Gaussian set. Use a pre-trained 2D depth estimation network to predict depth from the input images. Compute a loss between the rendered depth and the predicted depth to supervise geometry [S1].
4. **Optimization:** Optimize the Gaussian parameters (positions, scales, rotations, opacities, SH coefficients) using the depth loss and standard photometric rendering loss. The method may also remove Gaussians with incorrect depth values [S1].
5. **Regularization (Optional):** Utilize Score Distillation Sampling (SDS) loss to encourage fidelity in novel viewpoints, if applicable [S1].

**Formulas:**
- Depth Loss: $L_{depth} = || D_{rendered} - D_{predicted} ||$, where $D_{rendered}$ is the rendered depth and $D_{predicted}$ is the depth from the 2D network [S1].
- SDS Loss: $L_{SDS} = \nabla_{\theta} \mathbb{E}_{\epsilon, t} [w(t) (\epsilon_{\theta}(z_t, t, y) - \epsilon)]$ (Standard SDS formulation, cited as used in FSGS [S1]).

**Parameters:**
- Pre-trained 2D depth estimation network: Not specified which network (e.g., MiDaS, DPT) [S1].
- Number of unpooling iterations or expansion factor: Not given in the sources.

**Edge Cases and Pitfalls:**
- Sparse SfM initialization may have large gaps, requiring effective unpooling to avoid holes in the reconstruction [S1].
- Incorrect depth predictions from the 2D network can mislead optimization; the method mentions removing Gaussians with incorrect depth values [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Enables reconstruction from sparse views, a regime where standard 3DGS fails [S1].
- **Strengths:** First method to explore sparse-view reconstruction via Gaussian unpooling [S1].
- **Limitations:** Relies on SfM for initialization, which may be unreliable in textureless or repetitive regions. Requires a pre-trained 2D depth network [S1].

## Background
FSGS was introduced as the first method to explore reconstructing 3D scenes from sparse view input using Gaussian unpooling [S1]. It builds on the 3D Gaussian Splatting framework [Parent] and addresses the limitation of requiring dense input views. It is part of a family of sparse-view methods including SparseGS, CoherentGS, and DNGaussian, which also use depth priors [S1].

## Subtopics

### SparseGS
Uses monocular depth and diffusion priors for ~3-view input. Choose when fewer views are available and diffusion priors are acceptable [S2, unofficial].

### DNGaussian
Uses depth-normal regularizers. Choose when normal consistency is critical [S2, unofficial].

### InstantSplat
Pose-free and sparse via MASt3R/VGGT geometry priors, trains in seconds. Choose for speed and no pose annotations [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
