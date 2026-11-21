# Deep Learning and Neural Representations


## Overview
This family addresses the ill-posed inverse problem of tomographic reconstruction by replacing or augmenting traditional discrete voxel grids with neural networks, leveraging their expressive power to recover 3D density fields from sparse, noisy, or limited-angle projections [G]. It distinguishes itself from sibling methods like [Compressive Volume Rendering Recovery Methods](1915-compressive-volume-rendering-recovery-methods.md) and [Iterative Solvers](10781-iterative-solvers.md) by optimizing continuous function parameters (network weights) rather than discrete voxel values, enabling high-resolution reconstruction from minimal data and simultaneous self-calibration of geometric parameters [G].

## Shared principles
See [Tomographic reconstruction as inverse rendering](45-tomographic-reconstruction-as-inverse-rendering.md) for the shared differentiable forward projection, adjoint back-projection, and iterative optimization framework [G].

## Subtopics

### [Deep Learning and Neural Architectures](8282-deep-learning-and-neural-architectures.md)
This family encompasses methods that utilize deep neural networks to solve the inverse problem of tomographic reconstruction, mapping measured projection data (sinograms) directly or iteratively to a 3D volume estimate [8282-deep-learning-and-neural-architectures.md]. Unlike classical analytical methods (e.g., Filtered Back-Projection) or iterative algebraic techniques, these methods leverage the expressive power of neural architectures to regularize the ill-posed nature of the inverse Radon transform, particularly in sparse-view or noisy regimes [8282-deep-learning-and-neural-architectures.md]. They are distinguished from [Neural Radiance Fields and Implicit Neural Representations](13341-neural-radiance-fields-and-implicit-neural-representations.md) by their explicit focus on tomographic geometry (Radon transforms) and often their ability to produce explicit voxel-based or parametric density volumes suitable for quantitative analysis, rather than just view-dependent rendering [8282-deep-learning-and-neural-architectures.md].

### [Neural Radiance Fields and Implicit Neural Representations](13341-neural-radiance-fields-and-implicit-neural-representations.md)
This family applies Implicit Neural Representations (INRs) to tomographic reconstruction, replacing discrete voxel grids with continuous functions parameterized by neural networks [G]. It solves the ill-posed inverse problem of recovering 3D density fields from sparse, noisy, or limited-angle X-ray projections by leveraging the differentiability of the rendering pipeline [G]. Unlike traditional discrete methods, these approaches optimize network weights directly, allowing for high-resolution reconstruction from minimal data and enabling simultaneous self-calibration of geometric and photometric parameters [G].

### HybridCT
**Definition**: HybridCT is a computationally efficient method for practical cloud tomography that combines deep learning and linear algebra to reconstruct cloud extinction volumes from multi-angle imaging observations [S1, unofficial]. It combines a retrieval emulator with a reconstruction stage: (1) the retrieval is emulated using a deep learning U-Net architecture which is trained to map multi-angle radiance measurements into optical thickness values. (2) the reconstruction stage takes these multi-angle optical thickness values as input and uses SART (Simultaneous Algebraic Reconstruction Technique) to computationally reconstruct the cloud extinction volume [S1, unofficial].

**Steps**:
1. Emulate retrieval using a deep learning U-Net architecture trained to map multi-angle radiance measurements into optical thickness values [S1, unofficial].
2. Take the multi-angle optical thickness values as input and use SART to computationally reconstruct the cloud extinction volume [S1, unofficial].

**Parameters**: The method is demonstrated using synthetic multi-angle images representing the viewing angle geometry of the Multi-angle Imaging Spectroradiometer (MISR) onboard the Terra spacecraft at its red channel wavelength of 670 nm [S1, unofficial].

**Usage**: Designed for practical cloud tomography using synthetic cloud fields from Large Eddy Simulations and synthetic multi-angle radiance fields rendered using the 3D Monte Carlo radiative transfer model MYSTIC and the libRadtran library [S1, unofficial].

## Background
The method is used in practical cloud tomography to reconstruct cloud extinction volumes from multi-angle imaging observations [S1, unofficial]. Origin and history: Not given in the sources.

## Sources
[S1] lforster/hybridCT, https://github.com/lforster/hybridCT, fetched 2026-10-27 (unofficial)
