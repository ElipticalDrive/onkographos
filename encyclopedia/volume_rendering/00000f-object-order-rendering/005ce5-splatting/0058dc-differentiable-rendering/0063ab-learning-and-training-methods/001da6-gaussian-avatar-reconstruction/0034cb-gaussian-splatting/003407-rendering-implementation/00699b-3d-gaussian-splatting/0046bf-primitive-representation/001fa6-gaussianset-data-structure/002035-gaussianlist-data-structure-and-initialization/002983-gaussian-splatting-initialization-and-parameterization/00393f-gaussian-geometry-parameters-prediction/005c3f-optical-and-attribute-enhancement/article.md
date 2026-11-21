# Optical and Attribute Enhancement

Also called: Textured-GS, Attribute-Conditioned SH Enhancement, Defocus Blur Modeling

## Overview
Optical and Attribute Enhancement extends the standard 3D Gaussian Splatting representation by allowing spatial variation of color and opacity within individual Gaussian primitives, rather than treating them as uniform ellipsoids with fixed view-dependent color. It solves the limitation of low-order Spherical Harmonics (SH) in capturing high-frequency details and complex surface properties by introducing per-Gaussian texture maps and static attribute-conditioned offsets. It differs from standard 3DGS by evaluating SH basis functions on surface intersection vectors (Textured-GS) or baking MLP-predicted offsets into parameters (Attribute-Conditioned SH), and differs from naive blurring by selectively adjusting Gaussian scales based on defocus depth [S1].

## Shared principles
The family shares the goal of enhancing the visual fidelity of 3D Gaussians beyond the standard single-color, single-opacity model. All methods operate on the Gaussian primitives defined by position, rotation, scale, and base opacity. They modify the rendering or optimization pipeline to introduce higher-frequency detail or optical effects without necessarily increasing the number of primitives.

## Subtopics

### [Textured-GS](9436-textured-gs.md)
Textured-GS models spatially defined color and opacity variations across the Gaussian ellipsoidal surface using Spherical Harmonics (SH), evaluated against the surface normal rather than the viewing direction [9436-textured-gs.md]. It allows for deviations from standard ellipsoidal shapes, such as unidirectional opacity, and differs from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by using the surface intersection vector for SH evaluation [9436-textured-gs.md].

### [Attribute-Conditioned SH Enhancement](15410-attribute-conditioned-sh-enhancement.md)
This module recovers high-frequency visual details lost in low-order SH representations by predicting view-independent offsets via an MLP, which are statically baked into Gaussian parameters before inference [15410-attribute-conditioned-sh-enhancement.md]. It differs from dynamic neural view-dependent enhancement methods by introducing zero additional computational overhead during rendering [15410-attribute-conditioned-sh-enhancement.md].

### Defocus Blur Modeling
Models optical defocus by adjusting the scale $s$ of 3D Gaussians based on their distance from the camera's focal plane [S1]. During training, larger scale values $\hat{s}_j$ are used to rasterize blurred images, while smaller scale values $s_j$ are used during testing to render sharp details [S1]. Unlike a naive Gaussian kernel that blurs the entire image uniformly, this method selectively enlarges covariances only for Gaussians corresponding to blurred regions [S1].

## Background
The concept of modeling defocus blur using Gaussian scales originates from the observation that 3D Gaussians with greater dispersion (larger scale $s$) can represent scene points not located at the focal distance, while smaller scales model points at the focal distance [S1]. This approach was formalized in "Deblurring 3D Gaussian Splatting" [S1], which demonstrated that larger scale values are needed to adjust 3D Gaussians to rasterize blurred training images, whereas smaller-sized Gaussians are more suitable for modeling fine details in sharp testing images [S1]. The method avoids the limitations of naive Gaussian kernels, which blur the entire image and fail to model spatially varying blurriness [S1].

## Sources
[S1] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
