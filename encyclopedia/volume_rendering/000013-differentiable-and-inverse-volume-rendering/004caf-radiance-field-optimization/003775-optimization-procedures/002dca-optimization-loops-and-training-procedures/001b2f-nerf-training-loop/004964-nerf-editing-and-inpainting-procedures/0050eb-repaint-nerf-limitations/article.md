# RePaint-NeRF Limitations

Also called: NeRF editing constraints

## Overview
This article documents the limitations of the RePaint-NeRF framework, which performs NeRF editing via semantic masks and diffusion models [S1]. It addresses the constraints regarding computational efficiency, geometric fidelity during drastic shape changes, and view-angle-dependent artifacts. These limitations distinguish the method from sibling approaches that may prioritize speed or geometric consistency over the specific text-guided semantic editing pipeline [S1].

## Shared principles
Not given in the sources.

## Subtopics
Not given in the sources.

## Background
RePaint-NeRF was proposed by the authors of the paper "RePaint-NeRF: NeRF Editing via Semantic Masks and Diffusion Models" [S1]. The method utilizes a pre-trained 2D diffusion model to guide the regeneration of masked parts of a NeRF scene, incorporating background prompts and CLIP loss to handle invisible backgrounds, along with depth information for semantic mask acquisition [S1].

## Sources
[S1] RePaint-NeRF: NeRF Editting via Semantic Masks and Diffusion Models, https://doi.org/10.24963/ijcai.2023/201, fetched 2026-10-26
