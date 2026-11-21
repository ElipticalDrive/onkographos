# Blur and Transient Modeling


## Overview
This family extends 3D Gaussian Splatting (3DGS) to handle temporal dynamics, transient occluders, and real-world optical artifacts like motion and defocus blur [S1]. It distinguishes itself from sibling NeRF-based methods by relying on explicit Gaussian primitives rather than implicit neural fields, enabling efficient rendering without requiring explicit transient labels [S1].

## Shared principles
The family builds on the parent's mechanisms for temporal parameterization, canonical space mapping, and dynamic rendering [Parent](9265-dynamic-and-temporal-modeling.md).

## Subtopics

### [Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md)
NeTF is a neural modeling framework for Non-Line-of-Sight (NLOS) imaging that recovers the 5D transient function (spatial location and direction) implicitly represented by a Multi-Layer Perceptron (MLP) [4692-neural-transient-field-netf.md]. It solves the inverse problem of reconstructing hidden 3D scenes from sparse, unbalanced time-resolved measurements captured by Single-Photon Avalanche Diode (SPAD) arrays. Unlike sibling methods that use Cartesian rays, NeTF parametrizes input training data on spherical wavefronts and employs Markov chain Monte Carlo (MCMC) to handle sparse sampling, producing sharper edges in semi-occluded scenes. Choose it for NLOS reconstruction where sparse time-of-flight data must be mapped to 3D density volumes.

### [SWAG](6875-swag.md)
SWAG extends 3D Gaussian Splatting (3DGS) by conditioning Gaussians on learned appearance embeddings to handle dynamic scenes [6875-swag.md]. It specifically addresses transient occluders (e.g., pedestrians, moving vehicles) by training unsupervised transient Gaussians that learn to ignore these distractors, yielding state-of-the-art results on diverse photo scenes. It is distinguished from siblings like NeTF by its reliance on explicit Gaussian primitives with appearance conditioning rather than implicit neural fields. Choose it when reconstructing static scenes with moving occluders in standard RGB datasets.

### Blur Modeling in 3D GS
Techniques to explicitly incorporate blur modeling during training to address real-world capture issues like motion blur and defocus blur [S1]. It employs coarse-to-fine kernel optimization [S1] and photometric bundle adjustment [S1]. Choose it when standard 3DGS produces artifacts due to camera motion or defocus in real-world captures.

## Background
3D Gaussian Splatting (3DGS) excels on carefully curated datasets, but real-world captures often suffer from blurs such as motion blur and defocus blur [S1]. Recent approaches explicitly incorporated blur modeling during training to address this challenge [S1]. While the approximations made in GS contribute to its efficiency, they also underlie artifacts such as aliasing and blur, and limit the accuracy of illumination estimation and the modeling of challenging objects [S1]. Most existing efforts focus on specific failure modes rather than providing a universal solution, requiring practitioners to diagnose scene issues and apply targeted optimization strategies [S1].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
