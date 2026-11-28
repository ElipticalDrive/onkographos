# RePaint-NeRF Limitations
Also called: NeRF editing constraints

## Overview
This article documents the empirical limitations of the RePaint-NeRF method [S1], specifically regarding computational efficiency, geometric fidelity under significant shape changes, and view-angle consistency. It distinguishes these failure modes from the core optimization mechanics detailed in [RePaint-NeRF](14365-repaint-nerf.md) [S1].

## Shared principles

## Subtopics

### Computational Complexity
The method is both time and memory intensive [S1]. Generating a high-quality 3D model at the object level requires more than 30 minutes of optimization [S1]. Due to these resource demands, the method cannot scale to larger resolutions or larger scenes [S1].

### Geometric Fidelity
Training struggles when the target object shape differs drastically from the source object [S1]. In such cases, the edit may be incomplete, failing to fully replace the original geometry [S1].

### View-Angle Constraints
Restricted view angles cause geometric artifacts at image edges during optimization [S1]. Objects generated at the periphery may incorrectly combine features from multiple source objects, whereas central regions remain consistent [S1].

## Background
RePaint-NeRF was introduced in [S1] to enable NeRF editing via semantic masks and diffusion model guidance. The limitations described herein were identified during the evaluation of this framework, which uses gradient guidance from a pre-trained 2D diffusion model to regenerate masked parts of the scene [S1].

## Sources
[S1] RePaint-NeRF: NeRF Editting via Semantic Masks and Diffusion Models, https://doi.org/10.24963/ijcai.2023/201, fetched 2026-10-26
