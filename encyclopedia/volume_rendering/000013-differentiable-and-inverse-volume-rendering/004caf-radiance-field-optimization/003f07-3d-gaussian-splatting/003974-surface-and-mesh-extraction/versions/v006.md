# Surface and Mesh Extraction
Also called: Gaussian Surface Extraction, 3DGS Meshing

## Overview
This family comprises methods for deriving explicit geometric representations (meshes, segments) from implicit or volumetric 3D Gaussian Splatting (3DGS) models. It solves the problem that standard 3DGS produces a collection of opaque primitives rather than a watertight surface, enabling applications requiring collision detection, texturing, or semantic editing. It differs from sibling methods like [Gaussian Opacity Fields (GOF) Mesh Extraction](3692-gaussian-opacity-fields-gof-mesh-extraction.md) by offering distinct approaches to geometry definition: GOF discretizes opacity into a voxel grid for marching cubes, [Distance-based Gaussian Splatting](7411-distance-based-gaussian-splatting.md) aligns primitives to a pre-existing mesh via distance thresholds, and [2D-Lift-3D Scene Segmentation](11513-2d-lift-3d-scene-segmentation.md) assigns semantic labels via multi-view voting without explicit surface extraction.

## Shared principles
All methods operate on a trained set of $N$ 3D Gaussian primitives $\mathcal{G} = \{g_1, \dots, g_N\}$, where each $g_i$ is defined by a center $\boldsymbol{\mu}_i$, a covariance matrix $\boldsymbol{\Sigma}_i$ (often parameterized as rotation $\boldsymbol{R}_i$ and scale $\boldsymbol{S}_i$ such that $\boldsymbol{\Sigma}_i = \boldsymbol{R}_i \boldsymbol{S}_i \boldsymbol{S}_i^T \boldsymbol{R}_i^T$), an opacity $o_i$, and spherical harmonic coefficients $\mathbf{c}_i$ for color [S1, Parent]. The core challenge is mapping these volumetric, non-watertight primitives to a structured output (mesh vertices/faces or semantic labels). Common technical steps include:
1.  **Spatial Indexing:** Using KD-Trees or voxel grids to accelerate nearest-neighbor queries or density accumulation [S1, 3692].
2.  **Geometric Evaluation:** Computing distances between Gaussian centers and geometric entities (mesh faces, rays) using Mahalanobis distance or ray-primitive intersection [S1, 7411, 11513].
3.  **Thresholding/Classification:** Applying thresholds (e.g., density $T$, distance $d_{th}$, vote count $\tau$) to determine inclusion in the final structure [S1, 3692, 7411, 11513].

## Subtopics

### [Gaussian Opacity Fields (GOF) Mesh Extraction](3692-gaussian-opacity-fields-gof-mesh-extraction.md)
Extracts a triangulated mesh by discretizing the Gaussian opacity field into a voxel grid and applying marching cubes. Choose this when a watertight mesh is required from a trained model without retraining. It relies on opacity-based density accumulation rather than signed distance functions.

### [Distance-based Gaussian Splatting](7411-distance-based-gaussian-splatting.md)
Connects a pre-existing 3D mesh with Gaussian splats by classifying them as tightly-bound or loosely-bound based on distance to the mesh surface. Choose this when refining a known mesh geometry or pruning redundant volumetric data. It explicitly leverages pre-existing mesh geometry to align surface-aligned splats.

### [2D-Lift-3D Scene Segmentation](11513-2d-lift-3d-scene-segmentation.md)
Segments 3D Gaussian primitives by lifting 2D segmentation masks from reference views into 3D space using multi-view label voting. Choose this for partial scene editing or semantic decomposition. It operates directly on the volumetric primitive structure rather than extracting a surface mesh.

## Background
The extraction of explicit geometry from 3D Gaussian Splatting emerged as a necessary extension to the original rendering-focused formulation [Parent]. While 3DGS excels at novel view synthesis, its lack of explicit surface representation limits its use in geometry-aware applications. GOF introduced the concept of treating opacity as a volumetric density field for isosurface extraction [S1, 3692]. Distance-based methods were developed to integrate 3DGS with traditional mesh-based workflows, allowing for hybrid representations [S1, 7411]. Segmentation methods adapted 2D foundation models (like SAM) to the 3D Gaussian domain by propagating 2D labels through volumetric intersection tests [S1, 11513].

## Sources
[S1] Unofficial notes on Surface and Mesh Extraction methods for 3D Gaussian Splatting.
