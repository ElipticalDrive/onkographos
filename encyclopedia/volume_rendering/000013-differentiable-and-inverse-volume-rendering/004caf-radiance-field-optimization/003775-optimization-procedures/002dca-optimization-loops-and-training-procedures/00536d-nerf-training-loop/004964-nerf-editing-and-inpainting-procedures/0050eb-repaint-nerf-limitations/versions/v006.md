# RePaint-NeRF Limitations

Also called: NeRF editing constraints

## Overview
This article documents the specific failure modes and resource constraints of the RePaint-NeRF method [S1], which performs NeRF editing via semantic masks and diffusion model guidance. It addresses the limitations where the target object shape drastically differs from the source, restricted view angles cause geometric artifacts at image edges, and the process is computationally prohibitive for large scenes.

## Shared principles
Not given in the sources.

## Subtopics
Not given in the sources.

## Background
The RePaint-NeRF framework was proposed to edit NeRF content using text prompts by regenerating masked parts through gradient guidance from a pre-trained 2D diffusion model [S1]. It incorporates background prompts and CLIP loss to handle invisible backgrounds, and uses depth information for semantic mask acquisition [S1].

### Computational Cost
The method is described as time-consuming and space-consuming [S1]. Generating a relatively high-quality 3D model at the object level requires more than 30 minutes [S1]. Due to these resource demands, the method cannot be extended to larger resolutions of data or larger scenes [S1]. While other works have addressed time problems in similar frameworks like DreamFusion, the object-level generation time remains a bottleneck [S1].

### Shape Fidelity
The training process struggles when the new object shape is hugely different from the old one [S1]. In such cases, the model finds it difficult to complete the edit, leading to incomplete transformations [S1]. For example, changing a cup to a hotdog may result in an incomplete change [S1].

### View Angle Constraints
The shape of the generation is influenced by angle constraints [S1]. Generated objects at the periphery of the view may incorrectly combine features from multiple source objects [S1]. For instance, a generated daisy on the edge of the view might be composed of two orchids, whereas a daisy in the central location appears normal [S1].

## Sources
[S1] RePaint-NeRF: NeRF Editting via Semantic Masks and Diffusion Models, https://doi.org/10.24963/ijcai.2023/201, fetched 2026-10-26
