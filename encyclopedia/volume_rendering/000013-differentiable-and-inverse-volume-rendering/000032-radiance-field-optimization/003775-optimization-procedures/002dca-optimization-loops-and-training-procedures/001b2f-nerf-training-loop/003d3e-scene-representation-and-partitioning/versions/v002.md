# Scene Representation and Partitioning

Also called: Spatial Partitioning, Block-based NeRF, Distributed NeRF Partitioning

## Overview
Scene Representation and Partitioning addresses the scalability limits of global NeRF models, which suffer from high memory consumption, slow training convergence, and rendering artifacts when applied to large-scale environments or dynamic scenes [S1, S3]. By decomposing the 3D spatial domain into disjoint regions (blocks, tiles, or cells), each managed by a dedicated local neural network or data structure, these methods enable distributed training, faster inference, and the handling of scenes spanning thousands of square meters [S1, S3]. Unlike global approaches, partitioning allows for independent optimization of local regions, though it introduces challenges regarding boundary continuity and temporal consistency in dynamic settings [S3].

## Shared principles
All methods in this family rely on the decomposition of the scene volume $\mathcal{V}$ into $K$ non-overlapping sub-volumes $\{V_k\}_{k=1}^K$. Each sub-volume $V_k$ is associated with a local representation $\mathcal{M}_k$ (typically an MLP or a hybrid grid/MLP structure) that predicts density $\sigma$ and color $\mathbf{c}$ for points $\mathbf{x} \in V_k$.

The rendering process for a ray $\mathbf{r}$ involves identifying the intersection segments with each $V_k$. For a sample point $\mathbf{x}_i$ along the ray, the corresponding local model $\mathcal{M}_k$ (where $\mathbf{x}_i \in V_k$) is queried. The standard volume rendering integral is computed by aggregating contributions from these local queries:

$$
C(\mathbf{r}) = \sum_{i} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) \mathbf{c}_i
$$

where $\sigma_i = \sigma_k(\mathbf{x}_i)$ and $\mathbf{c}_i = \mathbf{c}_k(\mathbf{x}_i)$ if $\mathbf{x}_i \in V_k$. The transmittance $T_i$ accumulates across the entire ray, requiring careful handling of transmittance continuity at block boundaries [S3].

Training data is partitioned such that each $\mathcal{M}_k$ is trained only on pixels from training images whose rays intersect $V_k$ [S1, S3]. This reduces the training set size per module, enabling parallel training [S1].

## Subtopics

### [MegaNeRF](8580-meganerf.md)
MegaNeRF partitions the scene using spatial Voronoi cells with fixed centroids at a common altitude to handle large-scale static scenes ($10^5$ to $10^6$ m²). It differs from siblings by focusing on data parallelism for scalable training rather than just inference speed, and explicitly models foreground/background separation using an altitude-constrained ellipsoid. It is chosen when training on massive datasets with varying lighting where a single global model is infeasible.

### [BlockNeRF](8581-blocknerf.md)
BlockNeRF decomposes the scene into disjoint blocks, each with a dedicated small MLP, solving the problem of high-frequency information loss in global models for large environments. It sets itself apart by explicitly implementing appearance alignment between adjacent NeRFs to eliminate seams, a critical issue in naive partitioning. It is chosen for static, large-scale urban or architectural scenes requiring seamless visual integration.

### [NeRFPlayer](9440-nerfplayer.md)
NeRFPlayer extends partitioning to dynamic scenes by decomposing the spatiotemporal domain into stationary, deforming, and new area types using a decomposition field. It differs from siblings by explicitly modeling temporal dynamics and allowing selective skipping of expensive field queries based on area type probabilities. It is chosen for streaming dynamic scenes where efficiency and temporal coherence are paramount.

### [Spatial Partitioning for Distributed NeRF](10332-spatial-partitioning-for-distributed-nerf.md)
This method partitions 3D space into non-overlapping tiles by recursively splitting based on a PDF derived from a point cloud or ray samples, optimizing for cubic aspect ratios. Unlike BlockNeRF, which partitions based on camera visibility, this method partitions space directly, providing finer bounding boxes in dense regions. It is chosen for distributed training setups where workload balance and cubic tile shapes are desired.

## Background
The concept of partitioning NeRFs originated from the need to scale neural radiance fields to large environments. BlockNeRF introduced the divide-and-conquer strategy for large-scale scene representation [S3]. MegaNeRF extended this to support interactive fly-throughs and data parallelism for massive datasets [S1]. NeRFPlayer adapted these spatial concepts to handle temporal dynamics in streaming scenarios [S1]. Distributed partitioning methods further refined spatial splitting for multi-GPU training efficiency [S1, unofficial].

## Sources
[S1] MegaNeRF, BlockNeRF, NeRFPlayer, and Spatial Partitioning notes.
[S3] BlockNeRF notes.
