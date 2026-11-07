# Camera Pose and Parameter Optimization

Also called: Bundle Adjustment in NeRF, Joint Optimization of Poses and NeRF, NeRF with Unknown Poses

## Overview
This family of methods addresses the problem of jointly optimizing Neural Radiance Field (NeRF) network parameters and camera poses (intrinsics and extrinsics) when ground-truth camera parameters are unavailable, inaccurate, or require refinement. Standard NeRF training assumes fixed, known camera poses; these methods treat poses as differentiable trainable parameters within the optimization loop, enabling end-to-end bundle adjustment. They are distinguished from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by their explicit gradient updates to camera parameters, and from [Mip-NeRF 360](3306-mip-nerf-360.md) by their focus on pose uncertainty rather than angular resolution.

## Shared principles
All methods in this family share the core mechanism of treating camera parameters as learnable variables alongside NeRF weights.

**Parameterization:**
Camera poses are typically represented as rigid transformations in $SE(3)$. To ensure valid rotation matrices during gradient descent, poses are parameterized using:
1.  **Axis-Angle:** Rotation is represented by a 3D vector $\phi \in \mathbb{R}^3$, where the direction is the rotation axis and magnitude is the angle. The rotation matrix $R$ is computed via Rodrigues' formula [S1].
2.  **Lie Algebra ($se(3)$):** Poses are perturbed or optimized in the Lie algebra space. A 6D vector $v \in \mathbb{R}^6$ (3 for rotation, 3 for translation) is mapped to $SE(3)$ via the exponential map $\exp(v)$ [S1].

**Optimization Loop:**
The joint optimization minimizes a photometric loss $\mathcal{L}_{photo} = \| \hat{I} - I \|^2$ with respect to both NeRF weights $\Theta$ and camera parameters $\pi$. Gradients $\nabla_\Theta \mathcal{L}$ and $\nabla_\pi \mathcal{L}$ are computed via backpropagation through the volume rendering integral [S1].

**Challenges:**
The joint optimization landscape is highly non-convex. Small pose errors can cause large gradient mismatches, leading to divergence or convergence to incorrect local minima. Most methods employ coarse-to-fine strategies (e.g., frequency annealing, cascaded refinement) to stabilize convergence [S1].

## Subtopics

### [BARF (Bundle-Adjusting Neural Radiance Fields)](6872-barf-bundle-adjusting-neural-radiance-fields.md)
BARF reconstructs a NeRF from multi-view images when input camera poses contain errors. It addresses the sensitivity of standard NeRF to pose inaccuracies by jointly optimizing network weights and camera parameters using a coarse-to-fine refinement strategy that gradually introduces high-frequency positional encoding details. Unlike methods that rely on external priors, BARF directly regularizes the optimization landscape to prevent divergence.

### [NeRF with Unknown Camera Parameters](7405-nerf-with-unknown-camera-parameters.md)
NeRF with Unknown Camera Parameters (NeRF--) jointly optimizes camera parameters alongside a NeRF model on a sparse set of forward-facing views, eliminating the need for pre-estimated poses. It differs from baseline NeRF by treating camera intrinsics (shared focal length) and extrinsics (axis-angle rotation, translation) as differentiable trainable parameters.

### [L2G-NeRF](8671-l2g-nerf.md)
L2G-NeRF is a bundle-adjusting method that jointly optimizes neural 3D representations and camera poses by combining a local photometric alignment objective with a global geometric alignment objective. It differs from purely parametric methods (like BARF) by using a local-to-global process that is more robust to poor initialization, and from purely non-parametric methods by enforcing global geometric constraints via differentiable solvers.

### [NoPe-NeRF](8705-nope-nerf.md)
NoPe-NeRF trains NeRFs without precomputed camera poses by integrating undistorted monocular depth priors. It addresses the challenge of optimizing camera parameters and scene geometry simultaneously in challenging trajectories by generating a dense point cloud from a monocular depth estimation network and constraining relative poses between consecutive frames. Unlike methods relying solely on photometric consistency, NoPe-NeRF introduces a point cloud loss to stabilize optimization.

### [AltNeRF](8763-altnerf.md)
AltNeRF creates resilient NeRF representations from monocular videos without known camera poses. It solves the problem of joint optimization in highly non-convex landscapes by alternating between a Scene Prior Module (SPM) that provides robust depth and pose estimates, and a Scene Representation Module (SRM) that refines these estimates using NeRF. Unlike methods that strictly align rendered depth with priors, AltNeRF uses error-tolerant regularization to handle the imprecision of self-supervised depth estimates.

### [CBARF](9066-cbarf.md)
CBARF optimizes imperfect camera poses using a cascaded bundle-adjustment (BA) structure to progressively refine poses and rectify erroneous ones. It addresses the problem of joint optimization divergence caused by large initial pose errors, which standard methods like [BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md) may fail to resolve due to overfitting or local minima. By employing a coarse-to-fine scheduling strategy with distinct modulation factors, CBARF stabilizes the optimization landscape.

### NeRF Registration
#### definition
- [S1] A rigid NeRF registration method that minimizes a photometric loss between rendered images from two NeRFs, using VF and Normalizing Flows to guide ray sampling and generate novel views.
#### step
- [S1] Optimize a 6-DoF rigid transformation parameter (rotation and translation) to align the two NeRFs.
- [S1] Minimize a photometric loss (Equation 7 in the full paper) between rendered images from the aligned NeRFs.
- [S1] Use VF and Normalizing Flows to sample novel camera viewpoints and guide ray sampling during optimization.
#### parameter
- [S1] learnable scale factor (added to the 6-DoF parameters to handle unknown scale and illumination changes between scenes).
#### pitfall
- [S1] Photometric loss is susceptible to inaccuracies on textureless surfaces.
- [S1] High symmetry can cause convergence to a partially accurate solution that deviates by 180° from the true solution.
- [S1] Partial overlap or moving objects are not explicitly handled and may cause failure.

### Hybrid 2D/3D Registration Optimization Pipeline
#### definition
- [S2] A two-stage optimization pipeline for 2D/3D registration that first minimizes a differentiable network similarity loss ($L_N$) for global convergence, then switches to a traditional Grad-NCC loss ($L_{GNNC}$) for local refinement.
#### step
- [S2] Initialize pose parameters $\theta$. Optimize over $L_N$ using SGD with cyclic learning rate between $1e-6$ and $1e-4$ every 100 steps and momentum 0.9.
- [S2] Monitor convergence by calculating the standard deviation of $L_N$ over the last 10 iterations ($std_{L_N}$).
- [S2] If $std_{L_N} < 3 \times 10^{-3}$, switch optimization to Grad-NCC similarity.
- [S2] Optimize over $L_{GNNC}$ using SGD with cyclic learning rate between $1e-3$ and $3e-3$ and momentum 0.9.
- [S2] Stop when $std_{L_{GNNC}} < 1 \times 10^{-5}$.

### Progressive Camera Pose Generation
#### definition
- [S3, unofficial] A training strategy that gradually increases the angular distance of sampled target viewpoints from source views to stabilize optimization.
#### step
- [S3, unofficial] Sample noise uniformly from interval $[-\beta, +\beta]$ and add it to original Euler rotation angles of input view poses.
#### parameter
- [S3, unofficial] Parameter $\beta$: Grows linearly from 3 to 9 degrees throughout the course of optimization.

## Background
The problem of optimizing camera poses alongside NeRFs emerged as a solution to the dependency on accurate COLMAP reconstructions, which are computationally expensive and prone to failure in textureless or repetitive scenes. BARF [S1] was one of the first to introduce frequency annealing to decouple pose and geometry optimization. NeRF-- [S1] extended this to fully unknown intrinsics and extrinsics. Subsequent works like L2G-NeRF [S1], NoPe-NeRF [S1], AltNeRF [S1], and CBARF [S1] introduced various regularization strategies (geometric constraints, monocular depth priors, alternating optimization, cascaded BA) to improve robustness and convergence in challenging scenarios. NeRF Registration [S1] and Hybrid 2D/3D Registration [S2] apply similar joint optimization principles to aligning different scene representations or modalities. Progressive Camera Pose Generation [S3, unofficial] focuses on stabilizing novel view synthesis by controlling the difficulty of pose sampling.

## Sources
[S1] VF-NeRF: Viewshed Fields for Rigid NeRF Registration, https://doi.org/10.1007/978-3-031-73027-6_10, fetched 2026-10-26
[S2] Generalizing Spatial Transformers to Projective Geometry with Applications to 2D/3D Registration, https://arxiv.org/abs/2003.10987v1, fetched 2026-10-26
[S3] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
