# Hierarchical 3D Gaussian Splatting (H-3DGS)
Also called: H-3DGS, Hierarchical-3DGS
## Overview
Hierarchical 3D Gaussian Splatting (H-3DGS) is a method for large-scale 3D reconstruction that mitigates the memory bottlenecks of standard 3D Gaussian Splatting by training scene chunks independently and merging them into a global Level-of-Detail (LoD) hierarchy [S1, unofficial]. It sets itself apart from siblings like [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md) and [Octree-GS](7341-octree-gs.md) by using a unified training and rendering pipeline with external memory management to handle ultra-large scenes, whereas alternatives often rely on post-processing, manual threshold tuning, or different spatial partitioning strategies.
## Shared principles

## Subtopics

### CityGaussian / VastGaussian / DoGaussian
These methods combine chunked training with per-chunk LoD selection or progressive partitioning to address large-scale artifacts. They set themselves apart by decoupling appearance modeling or distributing training across nodes. Choose them when specific large-scale artifacts or appearance inconsistencies in chunked training need to be addressed beyond basic hierarchy management. See [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md).

### [Octree-GS](7341-octree-gs.md)
Octree-GS uses an octree structure to manage Gaussian primitives, offering a regular grid-based spatial hierarchy compared to the chunk-based approach of H-3DGS. It sets itself apart by achieving LOD effects in a single end-to-end training round using an accumulative LOD strategy. Choose it when a more regular, grid-based spatial hierarchy is preferred over arbitrary chunk boundaries. See [Octree-GS](7341-octree-gs.md).

### LightGaussian
LightGaussian focuses on compressing the Gaussian representation by pruning insignificant Gaussians and quantizing parameters, often used with chunked training. It sets itself apart by targeting storage size reduction rather than spatial hierarchy management. Choose it when the primary bottleneck is the storage size of the Gaussian attributes. See [LightGaussian](8036-lightgaussian.md).

### Scale-Adaptive Gaussian Splatting
This method adapts the scale of Gaussians dynamically during training to better fit scene geometry. It sets itself apart by focusing on the geometric fidelity of individual primitives rather than hierarchical organization. Choose it when high-frequency geometric details are critical. See [Scale-Adaptive Gaussian Splatting](8085-scale-adaptive-gaussian-splatting.md).

### Multi-scale filtering
Multi-scale filtering applies filtering operations at different scales to smooth or enhance the Gaussian representation, typically as a post-processing or regularization technique. It sets itself apart by operating on the primitive attributes rather than the spatial structure. Choose it when noise or artifacts in the reconstructed surface need to be reduced. See [Multi-scale filtering](8244-multi-scale-filtering.md).

### Virtual Memory for 3D Gaussian Splatting
This approach manages memory by swapping Gaussian data between VRAM and system RAM. It sets itself apart by handling data transfer logic independently of the training partitioning strategy. Choose it when the scene size exceeds available VRAM but system RAM is abundant. See [Virtual Memory for 3D Gaussian Splatting](8403-virtual-memory-for-3d-gaussian-splatting.md).

### Gaussian clustering and merging
This technique groups similar Gaussians to reduce the total primitive count. It sets itself apart by aggregating primitives based on similarity metrics rather than spatial distance. Choose it when the goal is to reduce the number of primitives for faster rendering or lower storage. See [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md).

### Iterative Gaussian Synopsis
This method creates a simplified synopsis of the Gaussian set iteratively. It sets itself apart by progressively discarding or merging primitives to meet a target complexity. Choose it for applications requiring real-time preview or low-bandwidth transmission. See [Iterative Gaussian Synopsis](9289-iterative-gaussian-synopsis.md).

### Budgeted Gaussian Splatting
This approach optimizes the Gaussian set under a strict memory or rendering budget. It sets itself apart by enforcing hard constraints on primitive count or memory footprint during optimization. Choose it when computational resources are severely constrained. See [Budgeted Gaussian Splatting](9672-budgeted-gaussian-splatting.md).

### Sequential Point Trees (SPTs)
SPTs are the underlying spatial data structure used in H-3DGS to manage the hierarchy, defining split points and levels of detail. They set themselves apart by providing a dynamic, distance-based partitioning mechanism. Choose them when a flexible, view-dependent hierarchy is required. See [Sequential Point Trees (SPTs)](10290-sequential-point-trees-spts.md).

### FilterGS
FilterGS applies filtering operations to Gaussian primitives to reduce artifacts and improve rendering quality. It sets itself apart by operating as a dedicated filtering stage within the splatting pipeline. Choose it when specific rendering artifacts require targeted suppression. See [FilterGS](12631-filtergs.md).

### Hierarchical 3DGS
Uses an octree-like tree of clusters; renders only the relevant LoD; achieves 99% reduction at far distances [S2, unofficial]. A cluster-based Level of Detail (LoD) method developed by Inria, capable of rendering large scenes like Mont Saint-Michel interactively [S2, unofficial].

### Varying Level of Detail (LoD) Noise
A technique to prevent overfitting of Gaussians to specific viewing distances by introducing noise into the hierarchy cut distances, ensuring coarser levels are trained occasionally [S1, unofficial]. For each training iteration, multiply the distances to the SPT centers by a factor calculated as $1 + 5^{u}$, where the exponent is a uniformly random variable $u \in (0, 1)$ [S1, unofficial]. The scaling factor applied to the distance is $1 + 5^{u}$, where $u \in (0, 1)$ is a uniformly random variable [S1, unofficial]. This ensures most iterations train near the highest LoD, but coarser levels are trained occasionally [S1, unofficial]. Without this noise, Gaussians of a specific scale overfit to views from a particular distance, leading to LoD popping artifacts [S1, unofficial].

### Varying Focal Length Adaptation
A method to account for differing focal lengths in datasets by using a relative distance metric for HSPT cuts, ensuring the required LoD is consistent across cameras with different projections [S1, unofficial]. Choose a base focal length $f_{\text{base}}$. For the current camera with focal length $f$, calculate the relative distance metric $\tilde{d}$ for HSPT cuts as $\tilde{d} = \frac{f}{f_{\text{base}}} d$, where $d$ is the actual distance [S1, unofficial]. Unlike standard methods that use only Euclidean distance, this method scales the cut distance by the ratio of the current focal length to a base focal length [S1, unofficial]. For example, a camera with double the focal length requires the cut distance to be halved to account for the larger projection footprint [S1, unofficial].

## Background
H-3DGS was introduced by Kerbl et al. [S1, unofficial]. It addresses memory bottlenecks in large-scale environments by dividing scenes into chunks for independent training before merging them into a global LoD hierarchy [S1, unofficial]. The implementation uses PyTorch and C++/CUDA, incorporating densification from Kheradmand et al. [2024] [S1, unofficial]. It is applied to ultra-large scale reconstruction, including aerial and street-level datasets [S1, unofficial].

## Sources
[S1] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
