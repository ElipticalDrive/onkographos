# Blur and Transient Modeling


## Overview
This family addresses artifacts arising from real-world capture conditions, specifically motion/defocus blur and transient occluders, which standard 3D Gaussian Splatting (3DGS) fails to model due to its instantaneous, static assumptions [S1]. It distinguishes itself from sibling dynamic methods (e.g., [D-NeRF](8660-d-nerf.md)) by focusing on per-frame or per-ray blur kernels and unsupervised transient entity separation rather than explicit temporal deformation fields or canonical space warping.

## Shared principles
The methods in this family share the following core mechanisms:

1.  **Artifact Decomposition:** The rendering pipeline is extended to separate or model non-ideal capture effects. For blur, the point-spread function (PSF) is integrated into the rasterization or optimization process [S1]. For transients, the scene representation is split into static geometry and transient occluders [S1, S2, unofficial].
2.  **Extended Gaussian Attributes:**
    *   **Blur:** Gaussians may be associated with blur kernels (e.g., Gaussian kernels) that are optimized during training to account for motion or defocus [S1].
    *   **Transients:** Two distinct sets of Gaussians are maintained: static Gaussians (with appearance embeddings) and transient Gaussians (standard parameters) [S1, S2, unofficial].
3.  **Joint Optimization:** The optimization objective includes photometric loss between rendered and ground truth images, but the gradient flow is structured to prevent static geometry from absorbing transient or blurred artifacts [S1, S2, unofficial].

## Subtopics

### [Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md)
[Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md) is a neural modeling framework for Non-Line-of-Sight (NLOS) imaging that recovers the 5D transient function (spatial location and direction) implicitly represented by a Multi-Layer Perceptron (MLP). Unlike [NeRF](52-name.md), which parametrizes input training data along lines (rays), NeTF parametrizes input training data on spherical wavefronts, formulating the NLOS process under spherical coordinates analogous to volume rendering under Cartesian coordinates. It is distinguished from sibling methods by its specific application to time-of-flight transient data and its use of Markov chain Monte Carlo (MCMC) to handle sparse and unbalanced sampling.

### [SWAG](6875-swag.md)
[SWAG](6875-swag.md) is a method for novel view synthesis in casual captures that extends 3D Gaussian Splatting (3DGS) by explicitly modeling transient occluders (e.g., pedestrians, vehicles) as unsupervised entities. It sets itself apart from standard 3DGS by conditioning the static scene Gaussians on learned appearance embeddings and training separate transient Gaussians to absorb photometric errors caused by moving objects, thereby preventing static geometry corruption.

### Blur Modeling in 3D GS
Techniques to explicitly incorporate blur modeling during training to address real-world capture issues like motion blur and defocus blur [S1]. The approach employs coarse-to-fine kernel optimization and photometric bundle adjustment to refine the blur parameters alongside the Gaussian attributes [S1].

## Background
3D Gaussian Splatting (3DGS) excels on carefully curated datasets but suffers from artifacts such as aliasing and blur in real-world captures [S1]. While approximations in GS contribute to efficiency, they limit the accuracy of illumination estimation and the modeling of challenging objects [S1]. Recent efforts have focused on specific failure modes, such as anti-aliasing, deblurring, or handling reflective/transparent objects, rather than providing a universal solution [S1]. In practice, one can diagnose issues (e.g., aliasing, blur, or reflective objects) and apply targeted optimization strategies [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
