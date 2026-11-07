# Scene Representation and Partitioning

Also called: Spatial Partitioning, Scene Decomposition

## Overview
Scene Representation and Partitioning addresses the scalability limits of single-NeRF models by decomposing large or dynamic scenes into localized sub-representations. It enables training and rendering of arbitrarily large environments or complex temporal dynamics by distributing computational load across multiple specialized networks, rather than relying on a single monolithic MLP. It differs from siblings like [Mip-NeRF 360](3306-mip-nerf-360.md) which focuses on multi-scale sampling within a single representation, and [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) which optimizes point density within a single model, by fundamentally altering the scene's structural representation to allow parallelism or dynamic handling.

## Shared principles
The core mechanism involves dividing the 3D spatial domain (and potentially the temporal domain) into disjoint or overlapping regions. Each region is assigned a dedicated neural network (MLP) or data structure (e.g., hash grid) that learns the local radiance field. During rendering, a ray is traced, and for each sample point, the system identifies the specific sub-representation responsible for that location and queries it. This decouples rendering time from the total scene size, as the query cost depends on the local complexity rather than the global scene volume.

## Subtopics

### [MegaNeRF](8580-meganerf.md)
A framework for large-scale static scenes using spatial Voronoi partitioning and data parallelism. It differs from siblings like [BlockNeRF](8581-blocknerf.md) by training submodules in parallel from scratch rather than decomposing a pre-trained model. It is chosen when interactive fly-throughs of thousands of images are required.

### [BlockNeRF](8581-blocknerf.md)
A method decomposing large-scale urban scenes into spatial blocks with separate NeRFs. It sets itself apart by using a 'total-part-total' approach to solve city-scale rendering, addressing high-frequency information loss in single models. It is chosen for static, large-scale urban environments requiring seamless blending.

### [NeRFPlayer](9440-nerfplayer.md)
A dynamic scene representation that decomposes spatiotemporal volumes into stationary, deforming, and streaming areas. It distinguishes itself by explicitly modeling temporal dynamics via a decomposition field, allowing sparse deformation and low-bitrate streaming. It is chosen for dynamic scenes with both static backgrounds and moving objects.

## Background
Originating from the need to scale NeRF beyond small-scale indoor scenes, this family of methods emerged around 2021-2022. MegaNeRF [S1] and BlockNeRF [S2] addressed the computational bottleneck of training single networks on massive datasets (e.g., city-scale). NeRFPlayer [S1] extended this partitioning concept to the temporal domain to handle dynamic scenes. These methods are used in large-scale reconstruction, autonomous driving simulations, and dynamic novel view synthesis.

## Sources
[S1] Mega-NeRF, NeRFPlayer, https://github.com/megvii-research/Mega-NeRF, https://github.com/NVlabs/NeRFPlayer, 2023-10-01
[S2] BlockNeRF, https://github.com/VisCom/BlockNeRF, 2023-10-01
