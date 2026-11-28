# GSN Model Architecture Components

Also called: Locally Conditioned Radiance Fields

## Overview
GSN decomposes a scene into many independent locally conditioned radiance fields to improve generative performance over global conditioning methods like GRAF [S1]. It distinguishes itself by using modulated linear layers in the generator and a local latent code representation $W \in \mathbb{R}^{c \times s \times s}$, whereas GRAF uses global conditioning with standard linear layers and $\pi$-GAN uses modulated sine layers [S1].

## Shared principles
The core mechanism of GSN is the decomposition of the scene representation into a grid of local radiance fields, each conditioned by a local latent code [S1].

**Local Latent Code Representation**
Instead of a single global latent vector $w \in \mathbb{R}^c$ that conditions the entire radiance field $f_\theta$, GSN uses a local latent grid $W \in \mathbb{R}^{c \times s \times s}$ [S1]. This decomposition allows the scene to be modeled as many independent locally conditioned radiance fields, which drastically improves generative performance (e.g., reducing FID from 68.42 to 41.75 and SwAV-FID from 6.14 to 4.14 in ablation studies) [S1].

**Local Coordinate System and Feature Sharing**
Each local radiance field operates within its own local coordinate system [S1]. This design enables feature sharing: a latent code $w_{ij}$ at grid position $(i, j)$ can represent the same semantic scene part regardless of its absolute position in the global grid [S1]. This robustness is validated by rigidly rearranging latent codes via 2D rotation; models with local coordinate systems maintain lower FID scores under rearrangement compared to global coordinate systems [S1].

**Camera Pose Normalization**
Camera poses are normalized relative to the middle frame of a sub-sequence during training [S1]. This enforces an egocentric coordinate system with the origin at the center of the scene [S1].

**Generator Network Architecture**
The radiance field generator in GSN utilizes modulated linear layers [S1]. This contrasts with GRAF, which uses standard linear layers, and $\pi$-GAN, which employs modulated sine layers [S1]. All compared models use the same base architecture and training pipeline to ensure fair comparison [S1].

## Subtopics

## Background
GSN was introduced in the paper "Unconstrained Scene Generation with Locally Conditioned Radiance Fields" [S1]. The method was developed to address limitations in global conditioning approaches for generative modeling of radiance fields [S1]. It was evaluated on unconstrained scene generation tasks where an interactive agent explores scenes, collecting RGB, depth, and camera pose data [S1]. Performance was measured using Fréchet Inception Distance (FID) and SwAV-FID metrics, sampling 5,000 real and 5,000 generated images for evaluation [S1].

## Sources
[S1] Unconstrained Scene Generation with Locally Conditioned Radiance Fields, https://doi.org/10.1109/iccv48922.2021.01404, fetched 2026-10-26
