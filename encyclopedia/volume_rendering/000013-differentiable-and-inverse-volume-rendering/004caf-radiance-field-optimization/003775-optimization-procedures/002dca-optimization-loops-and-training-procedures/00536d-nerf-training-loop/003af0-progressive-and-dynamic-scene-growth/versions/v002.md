# Progressive and Dynamic Scene Growth


## Overview
Progressive and Dynamic Scene Growth covers training procedures that iteratively expand the neural representation's capacity or the training dataset during optimization to solve the problem of reconstructing complex or large-scale scenes with limited initial data or fixed network topologies. This family sets itself apart from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) and [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md) by actively modifying the model's topology or supervision signals during training rather than relying on static priors or fixed sampling schedules [G].

## Shared principles

## Subtopics

### [Dynamic Growth Parameters](2968-dynamic-growth-parameters.md)
An adaptive training strategy that expands network capacity based on local scene complexity and prediction uncertainty. It distinguishes itself from static variants by coupling the network topology directly to the training error, growing $k$ new branches per step based on k-means clustering of high-uncertainty query points. It is chosen when a scene requires fine-grained detail recovery in specific regions without pre-defining the entire network hierarchy.

### [Progressive View Synthesis Training](8666-progressive-view-synthesis-training.md)
A training strategy for Panoramic NeRF that reconstructs a 3D scene from a single input panorama by progressively adding synthetic reference views to resolve occlusions and improve semantic coherence. It distinguishes itself by replacing generic depth constraints with a collaborative RGBD inpainting and progressive inpainting-and-erasing pipeline, explicitly synthesizing plausible content for occluded regions rather than relying on geometric priors or feature matching alone. It is chosen for single-view panoramic reconstruction where standard NeRF methods suffer from foggy effects or invalid floating densities.

### CityNeRF
A progressive learning paradigm that synchronously grows the NeRF model and training set. It distinguishes itself by starting with shallow basic blocks for distant views and continuously attaching new blocks as training iterates to adapt to far-near view details. It is chosen for large-scale urban reconstruction where scene complexity varies significantly across space. It effectively activates high-frequency channels in position encoding and unfolds more complex details as training progresses [S1].

## Background
The concept of progressive learning in NeRF was introduced by Xiangli et al. in 2021 with CityNeRF, which proposed synchronously growing the NeRF model and training set to handle large-scale urban environments [S1]. This approach was developed alongside related scene decomposition methods like Block-NeRF (Tancik et al., 2022), which uses a "total-part-total" method to decompose scenes into blocks for separate training and decouples rendering time from scene size [S1]. These methods are primarily used in large-scale urban reconstruction where scene complexity varies significantly across space [S1].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
