# Evaluation Pipeline

## Overview
The Evaluation Pipeline is a standardized protocol for assessing the geometric quality of 3D surface reconstructions from color and depth inputs, specifically computing Chamfer distance ($C_{\ell_1}$), Intersection over Union (IoU), Normal Consistency (NC), and F-score [S1]. It distinguishes itself from general mesh metrics by explicitly culling occluded and unseen geometry using ground-truth camera trajectories, ensuring that errors are only computed on regions visible in the input streams [S1]. This prevents error inflation from hidden surfaces or unpredictable network outputs in unseen areas [S1].

## How it works
**Inputs:**
1. Reconstructed mesh $M_{pred}$ [S1].
2. Ground-truth mesh $M_{gt}$ [S1].
3. Ground-truth camera poses (extrinsics and intrinsics) for the input color/depth frames [S1].
4. Input color/depth streams (to define the viewing frusta) [S1].

**Outputs:**
Scalar values for Chamfer distance ($C_{\ell_1}$), IoU, NC, and F-score [S1].

**Data Structures:**
*   **Mesh:** List of vertices $V$ and faces $F$ [G].
*   **Point Cloud:** List of 3D points $P$ and associated normals $N$ [G].
*   **Voxel Grid:** 3D grid with edge length $L_v = 5$ cm [S1].

**Steps:**
1. **Mesh Subdivision:** Subdivide all meshes (both $M_{pred}$ and $M_{gt}$) such that the maximum edge length is strictly less than $1.5$ cm [S1].
2. **Visibility Detection:** For each vertex $v$ in the meshes, determine if it is visible in at least one input camera frame using the ground-truth camera trajectory [S1].
3. **Culling:** Identify triangles with zero visible vertices [S1]. Remove these triangles from the meshes [S1]. This step excludes occluded geometry and regions outside the viewing frusta, which are either hidden in the ground truth or unpredictable by the network [S1].
4. **Point Cloud Sampling:** Sample the culled geometry to generate point clouds $P_{pred}$ and $P_{gt}$ [S1]. The sampling density is 1 point per cm² of surface area [S1]. Compute the surface normal for each sampled point [G].
5. **Metric Computation:** Compute error metrics on the resulting point clouds [S1].
    *   **Chamfer Distance ($C_{\ell_1}$):** Compute the Chamfer distance between $P_{pred}$ and $P_{gt}$ [S1].
    *   **Normal Consistency (NC):** Compute NC based on cosine similarity between normals of corresponding points in $P_{pred}$ and $P_{gt}$ [S1].
    *   **IoU and F-score:** Voxelize the reconstruction using voxels with an edge length of 5 cm [S1]. Apply a 5 cm threshold for F-score computation [S1]. Compute IoU and F-score based on the voxelized representations [S1].

**Parameters:**
*   Max edge length for subdivision: $< 1.5$ cm [S1].
*   Sampling density: 1 point/cm² [S1].
*   Voxel edge length: 5 cm [S1].
*   F-score threshold: 5 cm [S1].

**Edge Cases and Pitfalls:**
*   **Occluded Geometry:** Failing to cull occluded triangles inflates error metrics because the network cannot predict hidden surfaces, and ground-truth meshes may contain hidden geometry not present in the input views [S1].
*   **Unseen Regions:** Regions outside the viewing frusta must be excluded, as the network's output in these areas is unpredictable and not constrained by input data [S1].

## Subtopics

### [Uniform Sampling Evaluation](9611-uniform-sampling-evaluation.md)
Uniform Sampling Evaluation is a geometric evaluation protocol for surface reconstruction that eliminates vertex-density bias by uniformly sampling a fixed number of points from the reconstructed mesh surface within the ground truth (GT) crop volume. It ensures a strictly fair geometric comparison by decoupling the evaluation density from the mesh tessellation quality. Unlike legacy vertex-based metrics, which construct evaluation point clouds from mesh vertices and face centers, this method prevents scores from being artificially inflated by denser tessellations. It is primarily used for evaluating Gaussian-based surface reconstruction on the Tanks and Temples (T&T) and structured scan datasets.

### [Intersection over Union (IoU)](17963-intersection-over-union-iou.md)
Intersection over Union (IoU) evaluates the spatial alignment between predicted and ground-truth masks by aggregating confusion matrix counts. It operates on discrete masks or voxel grids rather than continuous surfaces, making it sensitive to spatial overlap errors in volume rendering evaluation pipelines.

### Local Mapping Quality Evaluation
Local Mapping Quality Evaluation assesses mesh reconstruction accuracy and optimization speed of hierarchical implicit grid representations against ground-truth meshes using Chamfer-L1 distance and F-score [S2, unofficial]. On ScanNet scene 0000, HIGS requires 1.58s with a Chamfer-L1 error of 4.04 cm and an F-score of 83.32%, while iSDF requires 67.71s with 4.77 cm error and 77.87% F-score, and Neural Points requires 57.87s with 12.23 cm error and 40.89% F-score [S2, unofficial].

### Surface Reconstruction Evaluation Protocol
Surface Reconstruction Evaluation Protocol is a standardized evaluation protocol for assessing surface quality from SDFs using geometric and normal consistency metrics [S3]. It uniformly samples $10^5$ points from both the ground-truth mesh and the reconstructed mesh [S3]. It computes Chamfer Distance (CD), F1 score, and Normal Consistency (NC) to assess geometric and normal accuracy [S3]. On the Thingi30 dataset at $32^3$ input, Ours+MC achieves CD: $1.351 \times 10^{-5}$, F1: 0.820, NC: 0.950 [S3]. At $128^3$ input, Ours+MC achieves CD: $0.638 \times 10^{-5}$, F1: 0.939, NC: 0.988 [S3]. The average runtime on Thingi30 at $128^3$ input is 2.778s for Ours+MC, compared to 1095.909s for RFTA and 56.663s for PoNQ [S3].

## Operating characteristics
*   **Time Complexity:** Dominated by visibility detection ($O(V \cdot C)$ where $V$ is vertices and $C$ is cameras) and nearest-neighbor search for Chamfer/NC ($O(N \log N)$ with spatial indexing) [G]. Voxelization is $O(N)$ [G].
*   **Memory Complexity:** $O(V + F + N)$ for storing mesh and point clouds [G]. Voxel grid is $O(L^3)$ where $L$ is scene extent in voxels [G].
*   **Strengths:** Fair comparison by ignoring unseen/occluded regions; robust to tessellation density via subdivision and uniform sampling [S1].
*   **Limitations:** Requires ground-truth camera poses for visibility culling; sensitive to voxel size choice for IoU/F-score [S1].
*   **When to use:** Evaluating surface reconstruction from RGB-D data where visibility is a key constraint [S1].
*   **When not to use:** When evaluating scene completion (including hidden geometry) or when ground-truth camera poses are unavailable [S1].

## Background
The pipeline was introduced in the context of Neural RGB-D Surface Reconstruction [S1]. It addresses the need for fair evaluation metrics that account for the inherent limitations of reconstruction from limited views, specifically by excluding regions that are not visible in the input data [S1]. It is used on datasets like Tanks and Temples and structured scan datasets [S1].

## Sources
[S1] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
[S2] HIGS: Hierarchical Implicit Grids for Joint Geometric and Semantic Scene Understanding, https://doi.org/10.48550/arxiv.2609.38620, fetched 2026-10-26 (unofficial)
[S3] SuperSDF: Sparse SDF Super-Resolution for Surface Extraction, https://doi.org/10.1145/3799902.3811176, fetched 2026-10-26
