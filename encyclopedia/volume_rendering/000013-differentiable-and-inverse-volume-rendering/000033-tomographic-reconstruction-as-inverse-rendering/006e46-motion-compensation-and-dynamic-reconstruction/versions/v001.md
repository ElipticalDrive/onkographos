# Motion Compensation and Dynamic Reconstruction

Also called: Dynamic Tomography, Motion-Aware Reconstruction

## Overview
This family addresses the degradation of tomographic reconstruction quality caused by object motion during data acquisition, which manifests as motion blur or streak artifacts. Unlike static reconstruction methods that assume a rigid volume, these methods jointly estimate the 3D volume structure and the temporal displacement field (or motion blur kernel) from time-resolved or coded projection data. They are distinguished from siblings like [Compressive Volume Rendering Recovery Methods](1915-compressive-volume-rendering-recovery-methods.md) by explicitly modeling the kinematic state of the object, and from [Differentiable CT Execution Model](3066-differentiable-ct-execution-model.md) by incorporating specific physical motion models (displacement fields or coded exposure integrals) into the differentiable forward operator.

## Shared principles
All methods in this family extend the standard inverse rendering formulation by introducing motion parameters $m$ into the forward projection operator $\mathcal{P}$. The optimization problem becomes:

$$
\min_{x, m} \mathcal{L}(\mathcal{P}(x, m), p) + \lambda R(x) + \mu R_m(m)
$$

where $x$ is the volume, $m$ represents motion (displacement field or blur code), and $R_m$ is a motion regularizer. The gradient with respect to the volume and motion parameters is computed via the chain rule through the differentiable forward projection $\mathcal{P}$ and its adjoint $\mathcal{P}^T$. The core challenge is the coupling between $x$ and $m$, often requiring staggered or alternating optimization strategies to decouple the volume update from the motion update [S1].

## Subtopics

### [Dynamic Tomographic Reconstruction with Motion Identification (DynTomo)](7376-dynamic-tomographic-reconstruction-with-motion-identificatio.md)
DynTomo is a staggered iterative method for reconstructing deforming 3D volumes from 2D projection data acquired over time, simultaneously identifying the full 2D space-time displacement field [7376-dynamic-tomographic-reconstruction-with-motion-identificatio.md]. It solves the problem of motion artifacts in tomography by alternating between algebraic volume reconstruction (DynART) and projection-based digital volume correlation (PDVC) for motion identification [7376-dynamic-tomographic-reconstruction-with-motion-identificatio.md]. Unlike standard Algebraic Reconstruction Techniques (ART) which produce blurry or split reconstructions under large displacements, DynTomo corrects for motion to yield sharp edges and accurate gray levels [7376-dynamic-tomographic-reconstruction-with-motion-identificatio.md]. It builds on [Tomographic reconstruction as inverse rendering](45-tomographic-reconstruction-as-inverse-rendering.md) by incorporating a parametric displacement model into the optimization loop [7376-dynamic-tomographic-reconstruction-with-motion-identificatio.md].

### [CodEx](18912-codex.md)
CodEx is a modular framework for joint temporal de-blurring and tomographic reconstruction that inverts motion blur introduced in sparse-view fly-scanning by encoding acquisition with a known binary code and using a non-convex Bayesian reconstruction algorithm [18912-codex.md]. It differs from linear deconvolution methods by handling sparse view sampling without aliasing issues, and differs from previous iterative methods like DynTomo by not requiring dense view samples [18912-codex.md].

### Warping Fields
#### definition
- [S1] A technique using a template RGBα volume $T(\mathbf{x})$ and a warp volume $W^{-1}(\mathbf{x})$ to alter effective voxel resolution and model motion, where $W^{-1}$ maps output positions to template sample locations.

#### step
- [S1] Compute final volume value: First evaluate the value of the inverse warp $W^{-1}(\mathbf{x})$, then sample the template volume $T$ at the resulting warped point. Formally:
$$
\mathbf{V}_{\text{RGB}\alpha}(\mathbf{x}) = \mathbf{T}_{\text{RGB}\alpha}(\mathbf{W}^{-1}(\mathbf{x}))
$$
where $\mathbf{x}$ is a 3D point in the output (rendering) space and the result is a 3D point in the RGBα template space.

#### pitfall
- [S1] Forward warps cannot represent resolution-increasing transformations without additional memory; inverse warps map small output areas to larger template areas, allowing higher resolution details while remaining well-defined everywhere for usable gradients.

## Background
The problem of motion artifacts in tomography has long been recognized, particularly in dynamic imaging scenarios such as cardiac CT or industrial fly-scanning. Traditional approaches often rely on gating or assuming rigid body motion. The integration of differentiable rendering with motion compensation allows for end-to-end learning of dynamic structures. The concept of using warping fields for dynamic volume representation was notably introduced in the context of neural rendering, where inverse warps are used to model non-rigid deformations and resolution changes [S1]. More recently, specific algorithms like DynTomo and CodEx have adapted these principles to physical tomographic reconstruction, addressing the ill-posed nature of joint volume-motion estimation through parametric basis functions and coded exposure models, respectively.

## Sources
[S1] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
