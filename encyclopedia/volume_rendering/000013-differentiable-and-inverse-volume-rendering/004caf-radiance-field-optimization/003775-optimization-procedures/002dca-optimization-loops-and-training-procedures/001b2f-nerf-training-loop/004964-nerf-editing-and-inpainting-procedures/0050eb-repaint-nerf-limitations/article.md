# RePaint-NeRF Limitations
Also called: NeRF editing constraints

## Overview
This article documents the specific limitations of the RePaint-NeRF method, which performs NeRF editing via semantic masks and diffusion model guidance [S1]. It addresses the high computational cost, geometric inconsistencies when target shapes diverge significantly from source geometry, and artifacts caused by restricted viewing angles [S1].

## Shared principles
Not given in the sources.

## Subtopics

### Computational Complexity
RePaint-NeRF is both time and memory intensive. The optimization process requires more than 30 minutes to generate a relatively high-quality 3D model at the object level [S1]. Due to these resource demands, the method cannot scale to larger resolutions or larger scenes [S1].

### Geometric Consistency
The training process struggles when the target object shape drastically differs from the source object [S1]. In such cases, the model may fail to completely replace the original geometry, leading to incomplete edits [S1].

### View Angle Constraints
Restricted view angles cause geometric artifacts, particularly at image edges [S1]. Generated objects at the periphery may incorrectly combine features from multiple source objects, whereas central regions remain normal [S1].

## Background
RePaint-NeRF was proposed in 2023 by the authors of "RePaint-NeRF: NeRF Editting via Semantic Masks and Diffusion Models" [S1]. It utilizes a pre-trained 2D diffusion model for gradient guidance to regenerate masked parts of a scene, incorporating background prompts and CLIP loss to mitigate invisible background issues [S1].

## Sources
[S1] RePaint-NeRF: NeRF Editting via Semantic Masks and Diffusion Models, https://doi.org/10.24963/ijcai.2023/201, fetched 2026-10-26
