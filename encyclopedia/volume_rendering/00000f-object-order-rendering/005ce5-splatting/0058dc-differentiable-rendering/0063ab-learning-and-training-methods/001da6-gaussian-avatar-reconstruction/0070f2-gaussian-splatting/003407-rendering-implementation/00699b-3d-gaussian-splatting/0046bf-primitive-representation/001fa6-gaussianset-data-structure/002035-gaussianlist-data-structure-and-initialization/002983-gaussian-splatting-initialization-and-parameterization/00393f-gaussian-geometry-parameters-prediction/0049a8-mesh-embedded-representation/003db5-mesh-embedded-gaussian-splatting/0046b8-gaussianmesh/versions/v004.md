# GaussianMesh

Also called: Mesh-GS, SuGaR Refinement

## Overview
GaussianMesh is a method that defines 3D Gaussians over an explicit triangle mesh to enable large-scale, interactive deformations driven by mesh editing [S1][S2, unofficial]. It solves the problem of coherent motion transfer in 3D Gaussian Splatting (3DGS) by tying Gaussian parameters to mesh geometry via joint refinement, where GS rendering guides adaptive splitting and mesh constraints regularize Gaussian shapes [S1]. It sets itself apart from parent methods by utilizing a reconstructed explicit mesh from calibrated images for initialization and employing specific face-split and normal-guidance strategies to handle large deformations without relying on implicit MLP-based skinning [S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A reconstructed explicit triangle mesh derived from calibrated images; 3DGS rendering parameters [S3].
*   **Outputs:** A set of 3D Gaussians embedded on the mesh faces, supporting real-time deformation [S3].

**Data Structures:**
*   **Exact Layouts:** Not given in the sources [S1][S2, unofficial][S3].
*   **Mesh:** Explicit triangle mesh with vertices and faces [S2, unofficial].
*   **Gaussians:** Each Gaussian is associated with a specific mesh face. Its position is determined by barycentric coordinates within that face, and its orientation/scale is influenced by the face's normal and local geometry [S1][S2, unofficial].

**Steps:**
1.  **Initialization:** Initialize Gaussians using the reconstructed explicit mesh derived from calibrated images [S3].
2.  **Adaptive Refinement:** Employ face split and normal guidance strategies to guide Gaussian learning and adapt density [S1, S3].
3.  **Regularization:** Introduce a regularization loss to preserve the shape of the Gaussians during optimization [S3].
4.  **Deformation:** Incorporate user-controlled mesh deformations [S3].
5.  **Rendering:** Achieve real-time deformation and rendering [S3].

**Formulas:** Not given in the sources [S1][S2, unofficial][S3].

**Parameters:** Not given in the sources [S1][S2, unofficial][S3].

**Edge Cases and Pitfalls:**
*   **Large Faces:** Exhibits artifacts when dealing with significant deformations in meshes with large faces, leading to inaccuracies [S3].
*   **Proxy Errors:** Relies on accurate semantic labels or mesh proxies; errors in segmentation or remeshing can easily produce artifacts [S1].

## Subtopics

### [OMeGa Joint Representation](15266-omega-joint-representation.md)
OMeGa is a joint representation framework that optimizes an explicit triangle mesh and 2D Gaussian splats simultaneously. It sets itself apart by decoupling geometric support from textural detail, allowing adaptive density control and free movement of splats within the face plane, unlike methods that allocate fixed splats or keep rotation/scale fixed. Choose it when accurate geometry in texture-less regions is needed and mesh geometry should guide splat optimization.

### [Gaussian Splitting for Bent Shapes](20251-gaussian-splitting-for-bent-shapes.md)
Gaussian Splitting for Bent Shapes is a procedure to split bent Gaussians into multiple well-formed ellipsoids to approximate non-rigid warping shapes. It sets itself apart by preventing visual artifacts caused by a single transformed Gaussian failing to represent bent shapes. Choose it when non-rigid warping causes single-Gaussian representation failures.

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources [S1][S2, unofficial][S3].
*   **Scaling:** Not given in the sources [S1][S2, unofficial][S3].
*   **Strengths:** Supports large, interactive deformations driven by mesh editing; enables real-time animation and scene editing [S1][S2, unofficial].
*   **Limitations:** Exhibits artifacts with significant deformations on meshes with large faces [S3]. Relies on accurate initial mesh proxies; errors in segmentation or remeshing cause artifacts [S1].
*   **When to use:** When large-scale, user-controlled mesh deformations are required for 3DGS avatars or scenes [S3].
*   **When not to use:** When the input mesh has large faces or when accurate semantic labels/proxies are unavailable [S1, S3].

## Background
GaussianMesh (also known as Mesh-GS) was introduced to realize large-scale 3DGS deformation [S3]. It utilizes a reconstructed explicit mesh from calibrated images to initialize Gaussians, employing face split and normal guidance strategies, and introduces regularization loss to preserve Gaussian shape [S3]. It builds on the concept of embedding Gaussians on meshes to disentangle motion from appearance [S1][S2, unofficial][Parent](9553-mesh-embedded-gaussian-splatting.md).

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Anttwo/SuGaR, https://github.com/Anttwo/SuGaR, fetched 2026-10-26 (unofficial)
[S3] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
