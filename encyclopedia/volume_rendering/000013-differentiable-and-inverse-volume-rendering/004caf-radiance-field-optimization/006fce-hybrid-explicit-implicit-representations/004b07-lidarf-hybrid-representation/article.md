# LidaRF Hybrid Representation

Also called: LidaRF

## Overview
LidaRF is a neural radiance field method for street scenes that fuses explicit LiDAR point cloud features with an implicit hash-based grid representation to improve geometry and rendering quality under sparse and collinear camera trajectories [S1]. It differs from prior LiDAR-only NeRF approaches by avoiding low-resolution and incomplete rendering caused by sparse LiDAR coverage and limited field of view; instead, it combines explicit LiDAR geometry with a high-resolution implicit grid [S1].

## How it works
**Inputs:**
* A set of LiDAR point clouds from all frames of a sequence [S1].
* Camera poses and intrinsics for the frames [S1].
* Image observations for color supervision [S1].

**Outputs:**
* Rendered images via standard volume rendering [S1].

**Data Structures:**
1. **Voxel Grid:** A sparse 3D grid constructed from aggregated LiDAR points. Each occupied voxel cell stores a 3-dimensional feature vector representing the averaged spatial position of all LiDAR points assigned to that cell [S1].
2. **LiDAR Embeddings ($P$):** A set $\{(p_i, f_i) \mid i = 1, \dots, N\}$, where $p_i$ is the 3D spatial position and $f_i$ is an $n$-dimensional neural feature vector extracted by a 3D sparse UNet [S1].
3. **Hash-based Feature Grid:** An implicit grid storing features for the background and regions not covered by LiDAR. The specific hash function and grid resolution are not detailed in the sources [S1].

**Algorithm Steps:**
1. **Aggregate LiDAR Data:** Aggregate LiDAR point clouds from all frames of the entire sequence to construct a denser point cloud set [S1].
2. **Voxelize:** Voxelize the aggregated point clouds into a voxel grid. For each voxel cell, average the spatial positions of all points falling into that cell to produce a single 3-dimensional feature vector per voxel [S1].
3. **Encode Geometry:** Encode the scene geometry features using a 3D sparse UNet operating on the voxel grid to capture local and global contextual information [S1]. The 3D sparse UNet takes the voxel grid along with its 3-dim features as input and outputs neural volumetric features [S1].
4. **Form LiDAR Embeddings:** Yield LiDAR embeddings $P = \{(p_i, f_i) \mid i = 1, \dots, N\}$, where each point $i$ is located at $p_i$ and associated with a vector $f_i$, which is the neural feature of the voxel cell it resides in, encoding the local and global geometry around $p_i$ [S1].
5. **Query and Fuse Features:** For a sample point $x$ along a ray, query its LiDAR feature if it resides within the voxel grid [S1]. Fuse the extracted LiDAR features with the hash-based feature grid [S1]. The exact fusion operation (e.g., concatenation or addition) is not specified beyond this step [S1].
6. **Decode:** Feed the hybrid features into MLPs to decode color $c$ and density $\sigma$ [S1].
7. **Render:** Apply standard volume rendering to generate the final image [S1].

**Formulas:**
* Volume rendering follows the standard discrete integral approximation: $C(r) = \sum_{i=1}^n T_i (1 - \exp(-\sigma_i \delta_i)) c_i$, where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance, $\sigma_i$ is the volume density, $\delta_i$ is the step length along the ray, and $c_i$ is the view-dependent color [G].

**Parameters:**
* **LiDAR Aggregation:** All frames of the sequence are used [S1].
* **Voxel Feature Dimension:** 3 dimensions (averaged spatial position) before UNet encoding [S1].
* **UNet Architecture:** 3D sparse UNet [S1].
* **Feature Dimension after UNet:** $n$ dimensions (not specified) [S1].

**Edge Cases and Pitfalls:**
* **Sparse LiDAR Coverage:** Relying on LiDAR features alone results in low-resolution rendering due to the sparse nature of LiDAR points [S1].
* **Limited Field of View:** LiDAR does not cover the entire scene (e.g., building surfaces above a certain height), yielding blank rendering in those regions if not fused with the implicit grid [S1].
* **Occlusion:** Depth supervision from LiDAR must account for occlusion [S1].

## Operating characteristics
* **Strengths:** Improves geometry and rendering quality in street scenes, particularly under sparse and collinear camera trajectories [S1]. Combines the geometric guidance of LiDAR with the completeness of a high-resolution implicit grid [S1].
* **Limitations:** Requires LiDAR data. Performance depends on the quality and density of the aggregated LiDAR point clouds [S1].
* **Complexity:** Not given in the sources.
* **Memory:** Not given in the sources.

## Background
LidaRF was introduced in the paper "LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes" [S1]. It addresses the limitations of previous LiDAR-only NeRF approaches, such as [9], which suffered from low-resolution rendering and incomplete coverage due to sparse LiDAR data and limited field of view [S1]. The method leverages denser LiDAR points accumulated across frames as depth supervision while accounting for occlusion, leading to extra losses $L_{ds}$ and $L_{aug}$ in addition to the original losses from Nerfactor $L_{nerfacto}$ [S1].

## Sources
[S1] LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes, https://doi.org/10.1109/cvpr52733.2024.01850, fetched 2026-10-26
