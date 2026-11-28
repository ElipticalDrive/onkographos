# Progressive and Dynamic Scene Growth

Also called: Dynamic Growth, Progressive View Synthesis, CityNeRF

## Overview
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details [S1]. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions [S1].

## Shared principles
The core mechanism across this family is the iterative expansion of the training scope. Whether expanding network branches based on uncertainty or adding synthetic views based on occlusion, the methods share a loop:
1.  **Evaluate:** Assess the current model's performance using a metric (uncertainty $\delta_i$ or occlusion masks).
2.  **Expand:** Modify the training configuration by adding new network components (branches) or new data sources (synthetic views).
3.  **Refine:** Jointly train the expanded model on the combined dataset.

Key shared terms include "growth step," "branch" (for network expansion), and "reference view" (for data expansion). The math for uncertainty-based growth involves clustering high-uncertainty points to determine branch centroids [S1]. For view-based growth, it involves projecting panoramic masks to perspective views for inpainting [S1].

## Subtopics

### [Dynamic Growth Parameters](2968-dynamic-growth-parameters.md)
An adaptive training strategy that expands network capacity based on local scene complexity and prediction uncertainty. It clusters high-uncertainty query points and grows parallel network branches to refine predictions in those regions, enabling coarse-to-fine representation without pre-defining the entire hierarchy.

### [Progressive View Synthesis Training](8666-progressive-view-synthesis-training.md)
A training strategy for Panoramic NeRF that reconstructs a 3D scene from a single input panorama by progressively adding synthetic reference views to resolve occlusions and improve semantic coherence. It distinguishes itself by replacing generic depth constraints with a collaborative RGBD inpainting and progressive inpainting-and-erasing pipeline, explicitly synthesizing plausible content for occluded regions rather than relying on geometric priors or feature matching alone.

### CityNeRF
A progressive learning paradigm that synchronously grows the NeRF model and training set. It starts by fitting distant views with shallow basic blocks. New blocks are continuously attached as training iterates to adapt to details in far-near views. This approach effectively activates high-frequency channels in position encoding and unfolds more complex details as training progresses [S1].

## Background
The concept of progressive learning in NeRF was introduced by Xiangli et al. in 2021 with CityNeRF, which proposed synchronously growing the NeRF model and training set to handle large-scale urban environments [S1]. This addressed the limitations of static models like Block-NeRF (Tancik et al., 2022), which decomposed scenes into blocks but required pre-defined hierarchies [S1]. Dynamic growth parameters and progressive view synthesis training emerged as specialized adaptations to handle uncertainty-driven complexity and single-view panorama reconstruction, respectively [S1]. These methods are primarily used in large-scale urban reconstruction and single-view synthesis tasks where data is sparse or scene complexity varies significantly across space [S1].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
