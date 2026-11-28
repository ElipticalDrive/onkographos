# Progressive and Dynamic Scene Growth

Also called: Hierarchical NeRF Allocation, Progressive Learning Paradigm

## Overview
Progressive and Dynamic Scene Growth refers to training procedures that adaptively increase model capacity or expand the training dataset during optimization, rather than using a fixed architecture and static data subset [S1][S2, unofficial][S3]. This family solves the problem of balancing computational resources with reconstruction fidelity in complex, large-scale, or continuously changing scenes [G]. It sets itself apart from static NeRF training by dynamically allocating representational power (via network branches, octree leaves, or block attachments) to regions of high uncertainty or novelty, and by incrementally incorporating new observations to prevent catastrophic forgetting or to handle unbounded data streams [S1][S2, unofficial][S3].

## Shared principles
The core mechanism across this family is the conditional expansion of the scene representation or training scope based on a metric of insufficiency (e.g., uncertainty, novelty, or temporal progression) [S1][S2, unofficial][S3].

1. **Metric Computation:** A scalar metric is computed for spatial regions or training samples. Common metrics include reconstruction uncertainty $\delta_i$ [S1], voxel novelty ratio $\tau_{spawn}$ [S2, unofficial], temporal thresholds [S2, unofficial], or inlier counts [S2, unofficial].
2. **Expansion Trigger:** When the metric exceeds a predefined threshold, a growth event is triggered.
    * *Network Growth:* New sub-networks (branches) are initialized at cluster centers of high-uncertainty points [S1].
    * *Structural Growth:* New octree leaves are spawned when the ratio of unmodeled visible voxels $|V_{new} \setminus V_{covered}| / |V_{new}|$ exceeds $\tau_{spawn}$ [S2, unofficial].
    * *Data Growth:* New camera views are added to the training buffer, often with specific initialization strategies for their poses [S2, unofficial][S3].
3. **Re-optimization:** The expanded system (new branches, leaves, or data points) is jointly optimized with the existing parameters. This often involves freezing older parameters for a duration $T_{freeze}$ to prevent disruption [S2, unofficial].
4. **Pruning/Frozen State:** To bound memory and computational cost, regions or parameters that are no longer active or have exceeded a temporal threshold are frozen or pruned [S2, unofficial].

## Subtopics

### [Dynamic Growth Parameters](2968-dynamic-growth-parameters.md)
Dynamic Growth Parameters is a training procedure that adaptively increases model capacity by adding network branches in regions of high reconstruction uncertainty. Unlike static architectures, it clusters high-error query points and spawns child networks to refine predictions locally, enabling efficient coarse-to-fine scene representation. It is chosen when a single global MLP cannot capture high-frequency details in complex scenes without excessive memory usage. See [Dynamic Growth Parameters](2968-dynamic-growth-parameters.md).

### [Progressive View Synthesis Training](8666-progressive-view-synthesis-training.md)
Progressive View Synthesis Training (PERF) is a training strategy for Panoramic Neural Radiance Fields (NeRF) that synthesizes novel views from a single input panorama by progressively adding reference views to improve geometry and semantic coherence in occluded regions. Unlike methods that rely on semantic regularization via CLIP or DINO features on perspective patches, PERF uses collaborative RGBD inpainting and an inpainting-and-erasing strategy to generate high-quality pseudo-ground truth for unseen areas, avoiding the "foggy effects" and redundant geometry common in other few-shot NeRF approaches. It is chosen for single-view or few-shot panoramic reconstruction. See [Progressive View Synthesis Training](8666-progressive-view-synthesis-training.md).

### [Incremental Registration](11341-incremental-registration.md)
Incremental Registration is a training strategy that registers camera poses sequentially following the temporal order of input images, leveraging the intuition that adjacent camera motions in videos are smooth. It solves the problem of non-convex pose optimization and local minima in joint pose and radiance field optimization by providing good initializations for new frames based on previously registered ones. Unlike global registration methods that optimize all poses simultaneously, this method builds the trajectory incrementally, freezing older parameters to prevent catastrophic forgetting and reduce computational cost. It is chosen for large-scale or video-based NeRF training where global optimization is intractable. See [Incremental Registration](11341-incremental-registration.md).

### [Online Incremental NeRF Optimization](12066-online-incremental-nerf-optimization.md)
Online Incremental NeRF Optimization addresses the problem of reconstructing 3D scenes from a continuous stream of incoming camera views, rather than a fixed, pre-collected dataset. It differs from offline methods like DVGO by updating the scene representation $\Phi$ dynamically as new observations arrive, enabling active exploration and real-time adaptation. This approach is essential for robotics and large-scale mapping where data volume is unbounded and memory constraints require discarding older frames. See [Online Incremental NeRF Optimization](12066-online-incremental-nerf-optimization.md).

### CityNeRF
CityNeRF is a progressive learning paradigm that synchronously grows the NeRF model and training set [S1]. It starts by fitting distant views with shallow basic blocks. New blocks are continuously attached as training iterates to adapt to details in far-near views. This method effectively activates high-frequency channels in position encoding and unfolds more complex details as training progresses [S1]. It is chosen for large-scale urban scene reconstruction where multi-scale details are critical.

## Background
The concept of hierarchical and progressive scene representation originated from the need to scale NeRF to large environments. NeRF-Nav introduced a hierarchical NeRF allocation using an octree structure, where the root maintains a coarse global representation and leaves represent high-resolution local fields, spawned based on exploration frontiers [S2, unofficial]. This approach reduced memory usage by 3.8× compared to a single global NeRF while maintaining rendering quality (PSNR within 0.3 dB) [S2, unofficial]. The spawning condition is defined as $Spawn(L_{new}) \Leftarrow |V_{new} \setminus V_{covered}| / |V_{new}| > \tau_{spawn}$, where $\tau_{spawn} = 0.3$ is the novelty threshold [S2, unofficial]. Leaves whose last observation exceeds a temporal threshold are frozen and optionally pruned to bound memory [S2, unofficial].

In the context of urban scenes, Xiangli et al. proposed CityNeRF in 2021, a progressive learning paradigm that synchronously grows the NeRF model and training set [S1]. It starts by fitting distant views with shallow basic blocks, and new blocks are continuously attached as training iterates to adapt to details in far-near views [S1]. This method effectively activates high-frequency channels in position encoding and unfolds more complex details as training progresses [S1]. These methods address the limitations of monolithic NeRF structures in handling large-scale, multi-scale data [S1][S2, unofficial].

Incremental registration strategies were developed to handle the non-convexity of joint pose and NeRF optimization [S2, unofficial]. By registering frames sequentially and freezing older parameters, these methods mitigate local minima associated with global optimization [S2, unofficial]. Online incremental methods extend this to continuous streams, crucial for robotics applications where real-time adaptation and memory management are required [S1].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S2] NeRF-Nav: Hierarchical Neural Radiance Fields for Real-Time Robot Navigation and Obstacle Avoidance, https://doi.org/10.21203/rs.3.rs-8887612/v1, fetched 2026-10-26 (unofficial)
[S3] Implicit vs. explicit a comparative survey on NeRF and 3DGS for large-scale aerial scene editing, https://link.springer.com/article/10.1007/s10791-026-10029-9, fetched 2026-10-26
