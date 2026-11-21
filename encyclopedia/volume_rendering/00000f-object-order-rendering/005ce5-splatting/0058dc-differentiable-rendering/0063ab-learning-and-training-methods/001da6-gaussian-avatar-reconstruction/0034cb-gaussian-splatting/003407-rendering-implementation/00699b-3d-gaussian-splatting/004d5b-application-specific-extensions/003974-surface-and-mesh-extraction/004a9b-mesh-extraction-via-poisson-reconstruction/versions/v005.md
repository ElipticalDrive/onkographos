# Mesh Extraction via Poisson Reconstruction

Also called: Poisson Mesh Extraction, Poisson Surface Reconstruction

## Overview
This method extracts a textured triangular mesh from an optimized 3D Gaussian Splatting (3DGS) scene by converting rendered 2D depth and normal maps into a 3D oriented point cloud, which is then reconstructed into a surface using Poisson surface reconstruction [S1][S2, unofficial]. It differs from implicit field-based siblings like [Gaussian Opacity Fields (GOF) Mesh Extraction](3692-gaussian-opacity-fields-gof-mesh-extraction.md) by relying on explicit geometric back-projection rather than density thresholding, and from [Distance-based Gaussian Splatting](7411-distance-based-gaussian-splatting.md) by not requiring distance-field regularization during training [S1].

## How it works
**Inputs:** An optimized set of 3D Gaussians $\mathcal{G} = \{G_i\}_{i=1}^M$, where each $G_i$ has position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and color attributes. A set of training views with known camera poses and intrinsics.

**Outputs:** A textured triangular mesh.

**Data Structures:**
1.  **Rendered Depth Map:** A 2D array $D \in \mathbb{R}^{H \times W}$ storing the median depth of splats projected to each pixel.
2.  **Rendered Normal Map:** A 2D array $N \in \mathbb{R}^{H \times W \times 3}$ storing the surface normal in camera space for each pixel.
3.  **Oriented Point Cloud:** A list of points $P = \{(\mathbf{x}_j, \mathbf{n}_j, \mathbf{c}_j)\}_{j=1}^K$, where $\mathbf{x}_j$ is the 3D position, $\mathbf{n}_j$ is the normal, and $\mathbf{c}_j$ is the color.

**Steps:**
1.  **Render Depth:** For each training view, project all 3D Gaussians to 2D. The projected 2D covariance is computed as $\mathbf{\hat{\Sigma}}_i = J W \mathbf{\Sigma}_i W^T J^T$, where $J$ is the Jacobian of the affine approximation of the projection and $W$ is the rotation part of the extrinsic matrix [S2, unofficial]. Render the depth map $D$ by computing the median depth value of all splats contributing to each pixel [S2, unofficial].
2.  **Derive Normals:** Convert the rendered depth map $D$ back into 3D space to derive the corresponding normal map $N$. This is typically done by computing the spatial gradient of the depth map and transforming it to world or camera space [S2, unofficial].
3.  **Back-Project to Point Cloud:** For each pixel $(u, v)$ in the training views:
    *   Retrieve the depth $d = D[u, v]$ and normal $\mathbf{n}_{cam} = N[u, v]$.
    *   Unproject the pixel coordinates and depth to obtain the 3D position $\mathbf{x}$ in world space.
    *   Transform the normal $\mathbf{n}_{cam}$ to world space.
    *   Retrieve the color $\mathbf{c}$ from the RGB image at $(u, v)$.
    *   Add $(\mathbf{x}, \mathbf{n}, \mathbf{c})$ to the point cloud $P$.
4.  **Poisson Reconstruction:** Input the oriented colored point cloud $P$ into the Poisson surface reconstruction algorithm [S1][S2, unofficial]. The algorithm solves for an implicit function whose gradient field best matches the normals in $P$, extracting the mesh as a level set [S1].

**Parameters:**
*   **Depth Level:** The depth of the octree used in Poisson reconstruction. Typical value: 9 [S1].
*   **Point Count:** The number of points in the input cloud. Typical value: 2 million [S1].

**Edge Cases:**
*   **Missing Depth/Normal:** Pixels with no splats or invalid depth must be handled (e.g., ignored or filled) before back-projection to avoid artifacts in the point cloud [S2, unofficial].
*   **Normal Consistency:** The normals derived from depth gradients must be oriented consistently with the camera view direction to ensure correct surface orientation in the Poisson solver [S2, unofficial].

## Subtopics

### Screened Poisson Equation Framework
A framework introduced by Mechrez et al. that effectively preserves structural fidelity while achieving stylization [S3]. It modifies the standard Poisson equation to better handle stylization tasks, though its specific application to 3DGS meshing is noted in the context of structural preservation [S3].

## Operating characteristics
*   **Time Complexity:** Dominated by the rendering of depth/normal maps for all training views and the octree construction in Poisson reconstruction. Not given in the sources.
*   **Memory Complexity:** Proportional to the number of training views and the resolution of the depth/normal maps, plus the memory for the 2 million point cloud and octree. Not given in the sources.
*   **Performance:** Produces high-quality textured meshes when the Gaussian scene is well-regularized (e.g., using depth and normal priors) [S1].
*   **Strengths:** Simple pipeline; leverages existing Poisson solvers; produces watertight meshes [S1][S2, unofficial].
*   **Limitations:** Relies on the quality of rendered depth/normal maps; sensitive to noise in depth gradients; requires sufficient training view coverage [S1][S2, unofficial].
*   **When to use:** When a textured mesh is required from a 3DGS model and training data includes depth/normal priors or reliable depth estimation is possible [S1][S2, unofficial].
*   **When not to use:** When real-time mesh extraction is needed or when training views are sparse/occluded [S1].

## Background
Originated from the application of Poisson surface reconstruction [18] to 3DGS scenes, as described in DN-Splatter [S1] and AtomGS [S2, unofficial]. The method was adopted to leverage the geometric regularization provided by depth and normal priors during 3DGS optimization, ensuring well-distributed Gaussian positions aligned with the scene surface [S1].

## Sources
[S1] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
[S3] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
