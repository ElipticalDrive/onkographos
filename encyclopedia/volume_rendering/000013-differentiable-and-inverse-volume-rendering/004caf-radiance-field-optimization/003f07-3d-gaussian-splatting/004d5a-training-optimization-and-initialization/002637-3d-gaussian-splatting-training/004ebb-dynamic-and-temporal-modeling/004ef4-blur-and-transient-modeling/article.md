# Blur and Transient Modeling


## Overview
This family addresses artifacts in 3D Gaussian Splatting (3DGS) caused by real-world capture imperfections, specifically motion blur, defocus blur, and transient occluders (e.g., pedestrians, vehicles) [S1]. It distinguishes itself from sibling methods like [D-NeRF](8660-d-nerf.md) or [SWAG](6875-swag.md) by focusing on explicit blur kernel optimization and unsupervised transient suppression rather than deformation fields or canonical space mapping. It builds on [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md) but treats blur and transients as static artifacts to be filtered or modeled, rather than dynamic scene components to be reconstructed over time.

## Shared principles
The shared principles across all methods in this family are:

1.  **Artifact Modeling:** Methods explicitly model the degradation process affecting the input images. For blur, this involves modeling the point spread function (PSF) or blur kernel [S1]. For transients, this involves identifying and isolating elements that do not belong to the static scene geometry [S1].
2.  **Optimization Integration:** The modeling is integrated into the training loop of the 3DGS representation. This often involves additional loss terms or modified rendering equations to account for the blur or transient presence [S1].
3.  **Static Scene Focus:** The primary goal is to recover the static scene geometry and appearance, treating blur and transients as noise or occlusions to be mitigated, rather than as dynamic content to be preserved [S1].

## Subtopics

### [Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md)
NeTF is a neural modeling framework for Non-Line-of-Sight (NLOS) imaging that recovers the 5D transient function (spatial location and direction) implicitly represented by a Multi-Layer Perceptron (MLP) [4692-neural-transient-field-netf.md]. Unlike [NeRF](52-name.md), which parametrizes input training data along lines (rays), NeTF parametrizes input training data on spherical wavefronts, formulating the NLOS process under spherical coordinates analogous to volume rendering under Cartesian coordinates [4692-neural-transient-field-netf.md]. It is distinguished from sibling methods by its specific application to time-of-flight transient data and its use of Markov chain Monte Carlo (MCMC) to handle sparse and unbalanced sampling [4692-neural-transient-field-netf.md]. Choose this method when dealing with NLOS data where sparse sampling and transient effects are critical.

### SWAG
SWAG extends 3D Gaussian Splatting (3DGS) by conditioning Gaussians on learned appearance embeddings to handle dynamic scenes [6875-swag.md]. It specifically addresses the problem of transient occluders (e.g., pedestrians, vehicles) in casual photo captures by training unsupervised transient Gaussians that are explicitly ignored during rendering [6875-swag.md]. Unlike sibling methods that rely on depth regularization or pre-trained feature spaces, SWAG achieves state-of-the-art results on diverse photo scenes by directly learning to suppress occluders through appearance conditioning [6875-swag.md]. Choose this method when reconstructing static scenes from casual photos containing transient occluders.

### Blur Modeling in 3D GS
Techniques to explicitly incorporate blur modeling during training to address real-world capture issues like motion blur and defocus blur [S1]. Employ coarse-to-fine kernel optimization [S1]. Employ photometric bundle adjustment [S1].

## Background
The challenge of reconstructing reflective and transparent objects and handling blur in general scenes remains significant [S1]. While 3DGS excels on carefully curated datasets, real-world captures often suffer from blurs such as motion blur and defocus blur [S1]. Recent approaches explicitly incorporated blur modeling during training, employing techniques such as coarse-to-fine kernel optimization [193] and photometric bundle adjustment [325] to address this challenge [S1]. The approximations made in GS contribute to its efficiency but also underlie artifacts such as aliasing and blur, limiting the accuracy of illumination estimation and the modeling of challenging objects [S1]. Most existing efforts focus on specific failure modes rather than providing a universal solution [S1]. In practice, one can first diagnose which issues are present in a scene (e.g., aliasing, blur, or reflective and transparent objects) and then apply targeted optimization strategies [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
