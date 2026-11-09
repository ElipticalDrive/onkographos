# Scene Representation and Partitioning

Also called: Large-scale NeRF partitioning, Distributed NeRF training, Dynamic NeRF decomposition

## Overview
Scene Representation and Partitioning addresses the scalability limits of monolithic Neural Radiance Fields (NeRFs) by decomposing the 3D scene volume into smaller, independent sub-volumes, each modeled by a separate local network. This family of methods enables the training and rendering of large-scale static scenes [S1, S3], dynamic scenes with temporal coherence [S1], and distributed multi-GPU training workloads [S1]. It differs from [Specific Model Architectures](8585-specific-model-architectures.md) by focusing on spatial decomposition rather than network topology changes, and from [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md) by explicitly managing data parallelism and boundary continuity across spatial blocks.

## Shared principles
The core mechanism involves partitioning the scene volume $\mathcal{V}$ into $K$ non-overlapping sub-volumes (cells, blocks, or tiles) $\{V_k\}_{k=1}^K$. Each sub-volume $V_k$ is associated with an independent local model $\mathcal{M}_k$ (typically an MLP or a grid-based structure) that predicts density $\sigma$ and color $\mathbf{c}$ for points $\mathbf{x} \in V_k$.

**Data Partitioning:**
Training data (pixels from images) is assigned to sub-volumes based on visibility. For each training image, rays are cast, and pixels are assigned to the training set of every sub-volume $V_k$ that the ray intersects. This ensures that each local model is trained only on data relevant to its spatial region [S1, S3].

**Rendering Integration:**
During inference, a query ray $\mathbf{r}$ is intersected with all sub-volumes. The ray is decomposed into segments $\{seg_k\}$ corresponding to the intersection with each $V_k$. These segments are sorted by distance from the ray origin. The volume rendering integral is computed by iterating through the sorted segments, maintaining a global accumulated transmittance $T$:
$$
T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
$$
$$
C(\mathbf{r}) = \sum_{i} T_i \left( 1 - \exp(-\sigma_i \delta_i) \right) \mathbf{c}_i
$$
where $\sigma_i, \mathbf{c}_i$ are obtained by querying the local model $\mathcal{M}_k$ corresponding to the segment containing sample point $\mathbf{x}_i$ [S1, S3].

**Boundary Continuity:**
A critical shared challenge is ensuring seamless integration at block boundaries. Transmittance must accumulate continuously across block boundaries to prevent visual artifacts. Some methods employ appearance alignment techniques to smooth color/density discontinuities between adjacent blocks [S3].

## Subtopics

### [MegaNeRF](8580-meganerf.md)
MegaNeRF is a framework for training large-scale static scenes (up to $1,300,000 \text{ m}^2$) using spatial Voronoi partitioning to enable data parallelism. It distinguishes itself by modeling the foreground as an ellipsoid and using altitude constraints to skip underground sampling. Unlike BlockNeRF, it focuses on scalable training for interactive fly-throughs.

### [BlockNeRF](8581-blocknerf.md)
BlockNeRF decomposes the scene into non-overlapping spatial blocks, each modeled by an independent small MLP. It emphasizes the "total-part-total" approach for city-scale rendering and ensures seamless integration between adjacent blocks via appearance alignment. Unlike MegaNeRF, it prioritizes inference scalability and boundary continuity for static urban environments.

### [NeRFPlayer](9440-nerfplayer.md)
NeRFPlayer extends spatial partitioning to dynamic scenes by decomposing the volume into stationary, deforming, and new areas using a decomposition field. It routes points to specialized local models (stationary MLP, deformation field, radiance field) to enable high-fidelity reconstruction of moving objects with low latency. Unlike the static methods, it explicitly models temporal dynamics.

### [Spatial Partitioning for Distributed NeRF](10332-spatial-partitioning-for-distributed-nerf.md)
This method partitions 3D space into non-overlapping tiles for distributed NeRF training by recursively splitting space based on a Probability Density Function (PDF) derived from a point cloud. It optimizes for cubic aspect ratios to balance GPU workloads. Unlike BlockNeRF, which partitions based on training cameras, this method partitions space directly, allowing for finer bounding boxes in regions with more content.

## Background
The need for scene partitioning arose from the memory and computational constraints of training monolithic NeRFs on large-scale datasets. Standard NeRF scales poorly with scene size, requiring excessive VRAM and training time [S1, S3]. Early large-scale approaches like BlockNeRF and MegaNeRF introduced spatial decomposition to decouple rendering time from scene size and enable parallel training [S1, S3]. Subsequent work extended these principles to dynamic scenes (NeRFPlayer) and distributed multi-GPU systems (Spatial Partitioning for Distributed NeRF) to address workload imbalance and temporal coherence [S1].

## Sources
[S1] MegaNeRF, BlockNeRF, NeRFPlayer, and Distributed NeRF source notes.
[S3] BlockNeRF and general NeRF principles source notes.
