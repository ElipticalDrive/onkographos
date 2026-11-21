# Scene Representation and Partitioning

Also called: Spatial Partitioning, Scene Decomposition, Block-based Representation

## Overview
This family of methods addresses the scalability limits of single-network NeRFs by decomposing the scene into multiple, smaller representations (submodules, blocks, or fields) that are optimized and queried independently. While the parent [Scene Representation and Architecture](11368-scene-representation-and-architecture.md) establishes the global mapping from 6D coordinates to radiance properties, these methods partition the domain to enable parallel training, reduce memory footprint, or handle dynamic content. They differ from siblings like [Specific Model Architectures](8585-specific-model-architectures.md) by focusing on the structural decomposition of the scene space rather than the internal topology of the neural network itself.

## Shared principles
The methods in this family share the mechanism of mapping 6D input coordinates (3D position $\mathbf{x}$ and 2D viewing direction $\mathbf{d}$) to 4D output values (density $\sigma$ and color $\mathbf{c}$) using a differentiable neural network. This allows the radiance field to be optimized through the volumetric rendering integral.

**Input Representation:**
Raw coordinates $\mathbf{x}$ and $\mathbf{d}$ are rarely used directly due to the spectral bias of MLPs, which struggle to fit high-frequency details. They are mapped to a higher-dimensional space using positional encoding $\gamma(\cdot)$ [S1][S3]. For a coordinate $\mathbf{x}$, the encoding includes the original value and sinusoidal functions at increasing frequencies:
$$
\gamma(\mathbf{x}) = (\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x}))
$$
where $L$ is the number of frequency bands [S1][S3].

**Network Topology:**
The canonical implementation uses two coupled Multi-Layer Perceptrons (MLPs): a density network $F_\sigma$ and a color network $F_c$ [S1][S3].
1.  **Density Network:** Takes the encoded position $\gamma(\mathbf{x})$ as input. It consists of $L$ hidden layers with ReLU activations and skip connections from the input to every 4th layer to preserve low-frequency information [S1][S3]. The output is the density $\sigma$.
2.  **Color Network:** Takes the encoded position $\gamma(\mathbf{x})$, the raw viewing direction $\mathbf{d}$, and the intermediate feature vector from the density network (usually the output of the last hidden layer before the density head) as input. It predicts the color $\mathbf{c}$ using a final ReLU or sigmoid activation [S1][S3].

**Rendering Integration:**
During training, rays are sampled at points $\mathbf{x}_i$ along the ray. The MLPs predict $\sigma_i$ and $\mathbf{c}_i$. The pixel color $C(\mathbf{r})$ is computed via volumetric rendering:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
and $\delta_i$ is the distance between samples [S1][S3].

## Subtopics

### [MegaNeRF](8581-meganerf.md)
MegaNeRF is a framework for training large-scale 3D scenes that support interactive human-in-the-loop fly-throughs, addressing challenges of modeling thousands of images with varying lighting, large model capacities, and fast rendering. It solves the infeasibility of training a single global NeRF on urban-scale datasets by decomposing the scene into spatial cells, each represented by a separate NeRF submodule. Unlike DeRF and KiloNeRF, which use spatial partitioning to speed up inference of a pre-trained single network, MegaNeRF uses spatial partitioning to enable data parallelism for scalable training, as obtaining a pre-trained NeRF for large scales is infeasible. It achieves a 40x speedup over conventional NeRF rendering while remaining within 0.8 dB in PSNR quality.

### [NeRFPlayer](9440-nerfplayer.md)
NeRFPlayer is a dynamic scene representation method that decomposes the scene into stationary, deforming, and new areas to enable efficient streaming and high-quality rendering of dynamic environments. It solves the problem of temporal coherence and storage efficiency in dynamic NeRFs by explicitly modeling the temporal evolution of scene components rather than treating all dynamic changes as new geometry. Unlike sibling methods like MegaNeRF or Spatial Partitioning for Distributed NeRF which focus on static or distributed static scenes, NeRFPlayer specifically addresses the spatiotemporal decomposition required for video sequences, achieving higher PSNR than HyperNeRF and significantly faster rendering than DyNeRF.

### [Spatial Partitioning for Distributed NeRF](10332-spatial-partitioning-for-distributed-nerf.md)
Spatial Partitioning for Distributed NeRF is a method to divide 3D space into non-overlapping tiles for distributed training, optimizing for cubic aspect ratios to balance computational load across multiple GPUs. Unlike Block-NeRF, which partitions based on training camera locations, this method partitions space directly using a Probability Density Function (PDF) derived from scene geometry, allowing for finer bounding boxes in regions with higher content density.

#### Large-scale Radiance Field Representation (Block & LOD)
- **Definition:** A representation for large scenes using multiple blocks and hierarchical Levels of Detail (LODs) to manage representational capacity and resource loading, addressing the limited resolution of single-model approaches [S1].
- **Training Stage:** The scene is uniformly partitioned into $K$ blocks $\{B_k\}_{k=1}^K$, each centered at $c_k=(x_k, y_k)$ on the xy plane (ground plane). Only the finest LOD is represented and optimized. For a point $\mathbf{p} = (p_x, p_y, p_z)$, the corresponding block $B_k$ is determined by its projected xy coordinates $\mathbf{p}_{proj} = (p_x, p_y)$:
  $$
  \mathbf{p} \in \mathcal{B}_k, k = \mathop{\arg \min}\limits_{k}\|\mathbf{p}_{proj} - \mathbf{c}_k\|_{\infty}
  $$
  Within block $k$, the components are: (1) $f^k$: Hash encoding + MLP decoder for attributes (density, diffuse color, specular feature); (2) $\Phi^k$: Tiny deferred MLP for view-dependent effects; (3) $\psi^k$: Proposal MLP for sampling [S1].
- **Rendering Stage:** Hierarchical LODs are used. $2 \times 2$ blocks are merged into one block between consecutive LODs. For LOD $l \in \{1, \dots, L\}$, there are $K/4^{l-1}$ blocks. Components per block $k$ at LOD $l$ are: (1) $f^{k,l}$: Baked texture query (sparse voxel + triplane) for opacity, diffuse color, specular features; (2) $\Phi^{k,l}$: Tiny deferred MLP for view-dependent effects; (3) $\psi^{k,l}$: Multi-level occupancy grid for sampling [S1].
- **Parameters:** $K$ is the total number of base blocks in the finest LOD [S1].

## Background
The need for scene partitioning arises from the memory and computational constraints of representing large-scale environments with a single neural network. Early NeRF implementations struggled with urban-scale datasets containing thousands of images [S1]. MegaNeRF introduced spatial Voronoi partitioning to enable parallel training of submodules [S1]. Spatial Partitioning for Distributed NeRF refined this by using geometry-based PDFs for balanced distributed training [S1]. NeRFPlayer extended decomposition to the temporal domain for dynamic scenes [S1]. The Block & LOD approach further optimizes resource loading by baking attributes into textures for rendering [S1].

## Sources
[S1] City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web, https://doi.org/10.1007/978-3-031-72970-6_22, fetched 2026-10-26
