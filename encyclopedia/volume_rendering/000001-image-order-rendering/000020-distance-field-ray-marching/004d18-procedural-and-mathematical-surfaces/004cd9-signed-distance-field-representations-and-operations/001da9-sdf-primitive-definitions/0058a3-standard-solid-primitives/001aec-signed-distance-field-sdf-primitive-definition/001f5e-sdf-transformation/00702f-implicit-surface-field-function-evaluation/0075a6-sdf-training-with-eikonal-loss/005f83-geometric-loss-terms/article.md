# Geometric Loss Terms

Also called: Geometry Loss, Combined Geometric Regularization

## Overview
Geometric Loss Terms refer to a composite regularization strategy used during the training of Signed Distance Field (SDF) networks to enforce geometric consistency, smoothness, and the recovery of thin structures. It solves the problem of shape-radiance ambiguity and vanishing gradients in implicit surface reconstruction by combining multiple geometric constraints into a single objective function. It sets itself apart from sibling methods by explicitly aggregating Eikonal, depth, normal, smoothness, and bubble losses, whereas siblings like [MLP Distillation](4829-mlp-distillation.md) focus on knowledge transfer, [SDDF with Ellipsoidal Priors and Neural Residuals](10638-sddf-with-ellipsoidal-priors-and-neural-residuals.md) on structural priors, [SDF-Instant-NGP Training](10640-sdf-instant-ngp-training.md) on acceleration structures, and [TetraSDF Training Procedure](13235-tetrasdf-training-procedure.md) on tetrahedral discretization.

## Shared principles
The family shares the principle of supervising the implicit function $d(x; \theta)$ using geometric priors derived from 3D point clouds, depth images, and surface normals. The core mechanism involves minimizing the deviation of the predicted SDF from ideal geometric properties: unit gradient magnitude (Eikonal), alignment with observed depth and normals, spatial smoothness of the gradient field, and the presence of zero-level sets near thin or missing geometry (Bubble).

The general form of the geometric loss is a weighted sum of individual terms:
$$ \mathcal{L}_{\text{geo}} = \lambda_1 \mathcal{L}_{\text{eikonal}} + \lambda_2 \mathcal{L}_{\text{depth}} + \lambda_3 \mathcal{L}_{\text{normal}} + \lambda_4 \mathcal{L}_{\text{smooth}} + \lambda_5 \mathcal{L}_{\text{bubble}} $$
where $\lambda_i$ are hyperparameters that may vary during training [S1].

## Subtopics

### [Sphere Loss](2750-sphere-loss.md)
Sphere Loss is a regularization term used during the pretraining phase to initialize the implicit surface representation as a sphere [2750-sphere-loss.md]. It minimizes the mean squared error between the network's predicted SDF values and the analytical SDF of a sphere with a specified radius. This method sets itself apart from sibling approaches by enforcing a simple analytic ground truth for initialization rather than using teacher-student distillation, ellipsoidal residuals, or hash-grid acceleration [2750-sphere-loss.md].

### [Bubble Loss for Thin Object Recovery](17090-bubble-loss-for-thin-object-recovery.md)
Bubble Loss is a geometric regularization term used to recover thin or small objects that are typically missed due to vanishing gradients [17090-bubble-loss-for-thin-object-recovery.md]. It addresses the limitation where standard SDF optimization deforms initial geometry to fit target surfaces, often failing to recover gradients for thin or small structures due to vanishing gradients and sparse sampling [17090-bubble-loss-for-thin-object-recovery.md]. Unlike standard training which samples uniformly or based on density, this method uses an error-guided sampling strategy to iteratively guide SDF growth from spatial points where missing thin segments meet the existing surface, utilizing 2D image information to identify error-prone areas [17090-bubble-loss-for-thin-object-recovery.md].

### Geometry Loss
Geometry Loss is the specific composite formulation defined in [S1], aggregating Eikonal, depth, normal, smoothness, and bubble losses.
- **Formula**: $\mathcal{L}_{\text{geo}} = \lambda_1 \mathcal{L}_{\text{eikonal}} + \lambda_2 \mathcal{L}_{\text{depth}} + \lambda_3 \mathcal{L}_{\text{normal}} + \lambda_4 \mathcal{L}_{\text{smooth}} + \lambda_5 \mathcal{L}_{\text{bubble}}$ [S1].
- **Eikonal Term**: $\mathcal{L}_{\text{eikonal}} = \sum_{x \in X} (\|\nabla d(x)\|^2 - 1)^2$, where $X$ is a minibatch of 3D points uniformly sampled in 3D space and nearby the surface [S1].
- **Normal Term**: $\mathcal{L}_{\text{normal}} = \sum_{r \in R} \|1 - \hat{N}(r) \cdot N(r)\|_1$, where $\hat{N}$ is the predicted normal and $N$ is the ground truth normal [S1].
- **Parameters**: $\lambda_1, \lambda_2, \lambda_3, \lambda_4, \lambda_5$ are hyperparameters for the geometry loss terms. Their values vary during training steps [S1].

## Background
The concept of combining multiple geometric losses for SDF training originates from the need to handle shape-radiance ambiguity and improve the reconstruction of complex indoor scenes. The specific composite formulation $\mathcal{L}_{\text{geo}}$ was introduced in the context of intrinsic indoor scene reconstruction and editing via raytracing in neural SDFs [S1]. The Eikonal regularization component builds on the foundational work of [7] (cited in [S1]), which established the unit gradient magnitude constraint for SDFs. The Bubble Loss component was introduced to address the vanishing gradient problem in sigmoid-based SDF training, specifically for recovering thin structures [S1]. The method is used in neural rendering pipelines that require high-fidelity geometric priors for material estimation and scene editing [S1].

## Sources
[S1] I 2 -SDF: Intrinsic Indoor Scene Reconstruction and Editing via Raytracing in Neural SDFs, https://doi.org/10.1109/cvpr52729.2023.01202, fetched 2026-11-01
