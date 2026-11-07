# 3D Evaluation Metrics
## Overview
3D Evaluation Metrics quantify the fidelity of reconstructed volumetric or surface data against ground-truth references [S1, S2, unofficial][S3]. While [PSNR Metric Calculation](2742-psnr-metric-calculation.md) focuses on pixel-wise photometric error in rendered images, this method extends evaluation to volumetric consistency and geometric accuracy [S1, S2, unofficial][S3]. It distinguishes itself by providing metrics for both volumetric reconstruction quality (PSNR, SSIM on 3D grids) and surface geometry (Chamfer Distance, Normal Consistency) [S1, S2, unofficial][S3].

## How it works
The method comprises two distinct evaluation pathways: volumetric reconstruction metrics and surface geometry metrics.

### Volumetric Reconstruction Metrics
These metrics evaluate the accuracy of the reconstructed volume data itself.

**Inputs:**
- Reconstructed volume $V_{rec}$: A 3D grid of values (e.g., density, color, or intensity).
- Ground-truth volume $V_{gt}$: A 3D grid of reference values.
- Both volumes must be aligned spatially and normalized to $[0, 1]$ [S1, unofficial].

**Outputs:**
- Mean PSNR (dB).
- Mean SSIM.

**Steps:**
1. **Normalization:** Ensure $V_{rec}$ and $V_{gt}$ are normalized to $[0, 1]$ [S1, unofficial].
2. **MSE Calculation:** Compute the Mean Squared Error averaged over all voxels, including background (zero-valued) voxels [S1, unofficial]:
   $$MSE = \frac{1}{N} \sum_{i=1}^{N} (V_{rec,i} - V_{gt,i})^2$$
   where $N$ is the total number of voxels in the grid [S1, unofficial].
3. **PSNR Calculation:**
   $$PSNR = 10 \log_{10}\left(\frac{1}{MSE}\right)$$
   This is equivalent to $10 \log_{10}(MAX^2 / MSE)$ with $MAX=1$ [S1, unofficial].
4. **SSIM Calculation:** Compute SSIM50 using a window size of 11 and Gaussian weighting ($\sigma = 1.5$) [S1, unofficial]. Average the windowed SSIM values to obtain the final score [G].
5. **Aggregation:** Compute metrics for axial, coronal, and sagittal views by iterating over the respective dimensions of the volumes, then print or return the mean PSNR and SSIM for each view [S2, unofficial].

**Edge Cases:**
- **Background Inclusion:** MSE must include background voxels. Excluding them biases the metric towards foreground objects only [S1, unofficial].
- **Held-out Voxels:** For fair comparison, metrics may be computed on a held-out subset of voxels (e.g., 5% randomly selected) rather than the entire volume [S1, unofficial].

### Surface Geometry Metrics
These metrics evaluate the geometric fidelity of a reconstructed surface against a ground-truth surface.

**Inputs:**
- Estimated surface $\chi_1$: A set of points (point cloud) or mesh vertices.
- Ground-truth surface $\chi_2$: A set of points or mesh vertices.

**Outputs:**
- Chamfer Distance (CD).
- Normal Consistency (NC).

**Steps:**
1. **Chamfer Distance (CD):** Compute the symmetric distance between point sets $\chi_1$ and $\chi_2$ [S3]:
   $$CD(\chi_1, \chi_2) = \frac{1}{|\chi_1|} \sum_{x \in \chi_1} \min_{y \in \chi_2} ||x - y||_2 + \frac{1}{|\chi_2|} \sum_{y \in \chi_2} \min_{x \in \chi_1} ||y - x||_2$$
   Lower values indicate better geometric alignment [S3].
2. **Normal Consistency (NC):** For each point $i$ on the surface, obtain the estimated normal $N_i$ and ground-truth normal $N_i^*$ [S3]. Compute the cosine similarity $\cos(N_i, N_i^*)$ [S3]. Calculate the mean angular error:
   $$NC = \frac{1}{N} \sum_{i=1}^{N} (1 - \cos(N_i, N_i^*)) \times 100\%$$
   where $N$ is the number of points on the surface [S3].

**Pitfalls:**
- **Point Cloud Sampling:** CD is sensitive to the density of points in $\chi_1$ and $\chi_2$. Uneven sampling can skew results; ensure uniform sampling or use mesh-based evaluation if available [S3].
- **Normal Orientation:** Normals must be consistently oriented (e.g., outward-facing) for NC to be meaningful. Inconsistent orientation leads to $\cos(N_i, N_i^*) \approx -1$, inflating the error [S3].

## Subtopics

### [Mesh Reconstruction Evaluation](2909-mesh-reconstruction-evaluation.md)
Mesh Reconstruction Evaluation quantifies the geometric fidelity of a reconstructed triangular mesh against a ground-truth reference mesh. It solves the problem of assessing how well a surface reconstruction method captures the true shape, topology, and detail of the underlying scene. Unlike radiance-field metrics that evaluate visual appearance, this method focuses strictly on spatial accuracy, distinguishing it from [PSNR Metric Calculation](2903-psnr-metric-calculation.md) which measures pixel-level photometric error.

### [Standard Rendering Metrics](3473-standard-rendering-metrics.md)
Standard rendering metrics quantify pixel-wise reconstruction error and perceptual fidelity to evaluate radiance-field optimizations. PSNR provides a cheap, decibel-based error measurement, while SSIM and MS-SSIM serve as structural companions, LPIPS offers learned perceptual similarity, and FID and KID evaluate generative distributional quality on rendered crops. These metrics focus on image-space fidelity, distinct from geometry metrics like Chamfer distance.

### Normal Consistency
Metric for 3D surface reconstruction measuring consistency between estimated and ground truth surface normals [S3]. Formula: $\text{NC} = \frac{1}{N} \sum_{i=1}^N (1 - \cos(N_i, N_i^*)) \times 100\%,$ where $N_i, N_i^*$ are estimated and ground truth surface normals for the $i$-th point, $\cos(N_i, N_i^*)$ is cosine similarity, and $N$ is the number of points [S3]. Parameters: $N_i, N_i^*$ are the normals; $N$ is the point count [S3].

## Operating characteristics
- **Time Complexity:**
  - PSNR/SSIM: $O(N)$ for MSE, where $N$ is the number of voxels [S1, S2, unofficial].
  - Chamfer Distance: $O(|\chi_1| \cdot |\chi_2|)$ for naive nearest-neighbor search [S3].
  - Normal Consistency: $O(N)$, where $N$ is the number of points [S3].
- **Memory Complexity:**
  - Requires storage of the full volume or point clouds [S1, S2, unofficial][S3].
- **Measured Performance:**
  - In microscopy applications, PSNR and SSIM are computed on volumes normalized to $[0, 1]$ [S1, unofficial]. Typical fitting settings include 20,000 max iterations and early stopping patience of 500 iterations [S1, unofficial].
  - Splat counts for evaluation can range from 1K to 2M, with geometric spacing (doubling) from 1K to 512K [S1, unofficial].
- **Strengths:**
  - Provides a comprehensive view of reconstruction quality, covering both photometric and geometric aspects [S1, S2, unofficial][S3].
- **Limitations:**
  - Background voxels significantly impact PSNR if not handled correctly (must be included) [S1, unofficial].
  - Chamfer Distance is sensitive to point cloud density and sampling uniformity [S3].
- **When to Use:**
  - Use volumetric metrics (PSNR/SSIM) when evaluating radiance fields or voxel grids [S1, S2, unofficial].
  - Use geometric metrics (CD/NC) when evaluating mesh or point cloud reconstructions [S3].

## Background
3D evaluation metrics have evolved alongside differentiable volume rendering and neural radiance fields. Early methods relied heavily on image-space metrics like PSNR and SSIM [S2, unofficial]. As the field moved towards explicit 3D representations (meshes, point clouds), geometric metrics like Chamfer Distance and Normal Consistency became standard for assessing surface fidelity [S3]. In scientific imaging, these metrics are applied to large-scale volumetric data with specific protocols like held-out voxel sampling and background inclusion [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S3] Self-Supervised Dual Contouring, https://doi.org/10.1109/cvpr52733.2024.00448, fetched 2026-10-26
