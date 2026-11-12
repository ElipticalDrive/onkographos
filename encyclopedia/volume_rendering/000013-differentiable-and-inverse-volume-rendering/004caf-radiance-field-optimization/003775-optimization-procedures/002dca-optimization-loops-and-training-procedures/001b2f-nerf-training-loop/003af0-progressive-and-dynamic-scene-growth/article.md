# Progressive and Dynamic Scene Growth

Also called: Dynamic Growth Parameters, Progressive View Synthesis Training, Online Incremental NeRF Optimization, CityNeRF, Incremental Registration

## Overview
Progressive and Dynamic Scene Growth refers to a family of training strategies that expand the capacity, data, or topology of a neural radiance field during optimization, rather than optimizing a static representation on a fixed dataset. This family solves the problems of representing complex, large-scale, or dynamic scenes with limited initial data, and mitigates issues such as "foggy" densities, floating artifacts, and catastrophic forgetting in online settings. It sets itself apart from static NeRF variants by coupling the network architecture or training data directly to the training error or temporal sequence, growing $k$ new branches per step based on uncertainty [S1], iteratively expanding reference views for panoramic scenes [8666-progressive-view-synthesis-training.md], or continuously updating the field as new camera views arrive [12066-online-incremental-nerf-optimization.md].

## Shared principles
The methods in this family share the principle of iterative expansion. Instead of a single monolithic optimization phase, the training loop includes a growth or integration step that modifies the state of the system.

1.  **Iterative Expansion:** The training process is divided into stages or steps where the model's complexity (number of parameters, branches, or feature grid size) or the dataset size increases.
2.  **Trigger Mechanism:** Growth is triggered by specific criteria:
    *   **Uncertainty/Error:** In dynamic growth, new branches are added where the current model exhibits high uncertainty or reconstruction error [2968-dynamic-growth-parameters.md].
    *   **Temporal/Sequential Order:** In incremental registration, new frames are added based on their position in a video sequence [S2].
    *   **View Synthesis Needs:** In progressive view synthesis, new views are synthesized and added to resolve occlusions or expand the field of view [8666-progressive-view-synthesis-training.md].
    *   **Online Acquisition:** In online optimization, new views are added as they are captured by an active agent [12066-online-incremental-nerf-optimization.md].
3.  **Re-optimization:** After each growth step, the model is re-optimized using the expanded set of parameters or data. This often involves resetting or adjusting learning rates to accommodate the new capacity without destroying previously learned information.

## Subtopics

### [Dynamic Growth Parameters](2968-dynamic-growth-parameters.md)
Dynamic Growth Parameters define the configuration for the iterative expansion of a neural radiance field's network architecture during training. This method solves the problem of representing complex scenes with a single static network by dynamically adding capacity where the current model exhibits high uncertainty. It sets apart from static NeRF variants by coupling the network topology directly to the training error, growing $k$ new branches per step based on k-means clustering of high-uncertainty query points [2968-dynamic-growth-parameters.md].

### [Progressive View Synthesis Training](8666-progressive-view-synthesis-training.md)
Progressive View Synthesis Training (PERF) is a training strategy for Panoramic Neural Radiance Fields (NeRF) that synthesizes novel views from a single input panorama by iteratively expanding the set of reference views [8666-progressive-view-synthesis-training.md]. It solves the problem of synthesizing occluded regions in panoramic scenes, where standard NeRF methods suffer from "foggy effects" or "floating" densities due to lack of semantic and geometric constraints. Unlike sibling methods that rely on semantic regularization via CLIP or DINO features, PERF distinguishes itself by using collaborative RGBD inpainting and a progressive inpainting-and-erasing strategy to generate high-quality pseudo-ground truth for occluded areas, ensuring semantic coherence and valid surface convergence [8666-progressive-view-synthesis-training.md].

### [Online Incremental NeRF Optimization](12066-online-incremental-nerf-optimization.md)
Online Incremental NeRF Optimization is a training paradigm where the radiance field representation is updated continuously as new camera views are captured, rather than optimizing a static dataset offline. It solves the problem of active 3D reconstruction in dynamic or large-scale environments where data arrives sequentially. Unlike offline methods such as DVGO, which optimize based on a fixed, pre-collected set of images, this method plans the next best views and perceives new observations based on the current scene representation, adding them to the training set incrementally [12066-online-incremental-nerf-optimization.md].

### CityNeRF
#### definition
- [S1] A progressive learning paradigm that synchronously grows the NeRF model and training set.
#### step
- [S1] Starts by fitting distant views with shallow basic blocks.
- [S1] New blocks are continuously attached as training iterates to adapt to details in far-near views.
- [S1] Effectively activates high-frequency channels in position encoding and unfolds more complex details as training progresses.

### Incremental Registration
#### definition
- [S2] A training strategy that registers frames incrementally following temporal order, leveraging the intuition that camera motions in videos are smooth and adjacent poses are close.
#### step
- [S2] At the start of training, jointly optimize NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ from the first two frames $\{I_1, I_2\}$.
- [S2] After every $k$ iterations, add a new frame $I_i$ and initialize its pose $P_i$ by the previous pose $P_{i-1}$.
- [S2] Freeze the learning rate on poses $\{P_i\}$ and NeRF $\Theta$ until all frames are registered. A learning rate decay schedule may be applied after all $N$ images are added.
#### parameter
- [S2] $k$: The iteration interval for adding new frames. Value: 1k iterations for CO3D/HO3D datasets; 500 iterations for LLFF dataset.
- [S2] Freeze duration: The number of iterations to freeze learning rates after registration. Value: 100k iterations for HO3D and CO3D; 30k for LLFF.

## Background
The concept of progressive training in NeRF has evolved to address scalability and data efficiency. CityNeRF was proposed in 2021 by Xiangli et al. to handle large urban environments by synchronously growing the model and training set, starting with distant views and adding detail iteratively [S1]. This approach activates high-frequency channels in position encoding to unfold complex details [S1].

Incremental Registration was introduced in the ICON paper (2024) to tackle the non-convexity of joint pose and NeRF optimization. By leveraging the smoothness of camera motion in videos, it registers frames sequentially, initializing new poses from previous ones and freezing parameters to avoid local minima and overlapping registration errors [S2].

Dynamic Growth Parameters, as described in the notes, address the limitation of static network capacity by clustering high-uncertainty points to grow new branches, a technique associated with Recursive NeRF [2968-dynamic-growth-parameters.md].

Progressive View Synthesis Training addresses the specific challenges of panoramic NeRFs, such as distortion and occlusion, by using inpainting to generate pseudo-ground truth for new views [8666-progressive-view-synthesis-training.md].

Online Incremental NeRF Optimization extends these ideas to active reconstruction, where the system decides which views to capture next to maximize information gain, updating the field continuously [12066-online-incremental-nerf-optimization.md].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S2] ICON: Incremental CONfidence for Joint Pose and Radiance Field Optimization, https://doi.org/10.1109/cvpr52733.2024.00517, fetched 2026-10-26
