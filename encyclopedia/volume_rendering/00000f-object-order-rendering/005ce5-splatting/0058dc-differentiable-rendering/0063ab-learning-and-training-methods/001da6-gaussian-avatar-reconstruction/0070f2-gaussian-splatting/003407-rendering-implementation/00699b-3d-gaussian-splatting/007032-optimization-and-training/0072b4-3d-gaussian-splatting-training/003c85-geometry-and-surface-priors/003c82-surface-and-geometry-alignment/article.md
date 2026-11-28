# Surface and Geometry Alignment

Also called: Geometric Priors in Gaussian Splatting, Surface-Aligned Gaussian Optimization

## Overview
This family of methods addresses the lack of explicit, watertight geometry in standard 3D Gaussian Splatting (3DGS), which consists of floating, semi-transparent ellipsoids . By aligning the discrete Gaussian representation with continuous geometric priors—such as signed distance fields (SDFs), metric depth maps, or explicit meshes—these techniques enforce positional and normal consistency to produce high-fidelity surface representations . Unlike sibling methods that primarily focus on training-time regularization or post-hoc mesh extraction, this family encompasses a spectrum from re-initialization strategies and physically-based inverse rendering to contrastive token alignment, enabling applications ranging from editable mesh generation to accurate material decomposition .

## Shared principles
All methods in this family align the discrete Gaussian representation with a continuous geometric prior . The core mechanism adds geometric loss terms to the standard photometric rendering loss $\mathcal{L}_{rgb}$ . These losses enforce two properties: positional consistency, where Gaussian centers $\mathbf{c}_i$ are encouraged to lie on the zero-level set of a distance function (SDF or UDF) or align with metric depth estimates , and normal consistency, where the orientation of the Gaussians (derived from their covariance matrix or rotation quaternion) is aligned with the surface normal estimated from the geometric prior . The total loss function generally takes the form:
$$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{geo} \mathcal{L}_{geo} + \lambda_{norm} \mathcal{L}_{norm} $$
where $\mathcal{L}_{geo}$ is the geometric consistency loss and $\mathcal{L}_{norm}$ is the normal consistency loss . The weights $\lambda_{geo}$ and $\lambda_{norm}$ are hyperparameters that balance photometric fidelity against geometric accuracy [G].

## Subtopics

### [SuGaR](9191-sugar.md)
A post-hoc method that converts the discrete, volumetric 3DGS representation into a continuous, editable triangle mesh by aligning Gaussians with a reconstructed surface. It solves the problem that vanilla 3DGS produces floating ellipsoids with no explicit geometry, making it difficult to use in standard DCC pipelines for animation or physics. Unlike sibling methods that modify the training loop or use implicit fields, SuGaR operates after training, using a self-regularization loss to flatten Gaussians onto a surface and then extracting a mesh via Poisson reconstruction.

### [Surface-guided Gaussian Re-initialization](9199-surface-guided-gaussian-re-initialization.md)
A training strategy for 3DGS that resolves the unbalanced aggregation of Gaussians caused by densification and splitting operations. It makes trained Gaussians better fit the input monocular video by anchoring them to a surface representation. It is distinct from [SuGaR](9191-sugar.md) and [Geo-Splatting](9201-geo-splatting.md) by specifically targeting the re-initialization phase to correct aggregation artifacts rather than just loss regularization or initial placement.

### [Geo-Splatting](9201-geo-splatting.md)
A physically-based inverse rendering method that augments 3DGS with explicit geometric priors to improve normal estimation and light transport modeling. It solves the problem of inaccurate material decomposition and flawed relighting caused by approximated normals and semi-transparent Gaussian points in existing methods. Unlike siblings that rely on implicit constraints or low-frequency spherical Gaussians, Geo-Splatting differentiably constructs surface-aligned Gaussian points from an optimizable explicit mesh, leveraging well-defined mesh normals and opaque surfaces for superior decomposition and relighting quality.

### [GeoCE Loss](17334-geoce-loss.md)
A geometric regularization term used in ZipSplat to align token similarity with 3D proximity, ensuring that clustering merges image patches observing the same surface rather than grouping them by viewpoint. It solves the problem where feature-space merging does not guarantee geometric consistency, explicitly shaping the token space so that clusters correspond to physical surfaces. Unlike [Cone Loss](17337-cone-loss.md), which anchors individual tokens to their source patches, GeoCE handles token merging by enforcing that similar tokens are also 3D-proximate.

## Background
The standard 3D Gaussian Splatting method  provides fast, high-quality novel view synthesis but lacks an explicit surface representation, consisting instead of a collection of anisotropic Gaussians. This limitation hinders applications requiring watertight meshes, such as physics simulation, texturing, and animation. Early attempts to extract geometry from 3DGS relied on post-processing techniques like Poisson reconstruction on the Gaussian centers, often resulting in noisy or incomplete meshes due to the volumetric nature of the representation . To address this, subsequent research introduced geometric priors during the optimization process. Methods like NeuSG and GaussianPro integrated implicit neural fields or depth priors to guide the Gaussian placement . More recent approaches, such as Geo-Splatting, have moved towards explicit mesh-based proxies for physically-based rendering, while others like SuGaR have refined post-hoc alignment to produce clean, editable meshes . The introduction of contrastive losses like GeoCE further refined the geometric consistency of token-based Gaussian representations, ensuring that learned features correspond to physical surfaces rather than viewpoint-dependent artifacts .

