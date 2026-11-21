# Evaluation Pipeline

Also called: Reconstruction Evaluation, Visible Surface Evaluation

## Overview
This pipeline computes Chamfer distance ($C_{\ell1}$), intersection-over-union (IoU), normal consistency (NC), and F-score on reconstructed surfaces that are visible in the input color/depth streams [S1]. It differs from standard mesh evaluation by explicitly culling occluded or unseen geometry to prevent error inflation from regions where the network output is unpredictable or where ground truth contains hidden surfaces [S1]. It builds on [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md) but restricts the comparison domain to the viewing frusta of the input cameras.

## How it works
**Inputs:**
- Reconstructed mesh $M_{rec}$ (triangle soup or manifold mesh).
- Ground truth mesh $M_{gt}$ (optional, depending on metric; primarily used for visibility definition in this specific pipeline context, though IoU/F-score often compare $M_{rec}$ to $M_{gt}$).
- Camera poses $P_i$ (extrinsics) and intrinsics $K_i$ for all input frames $i=1 \dots N$.
- Input color/depth streams (used to define the "visible" domain).

**Outputs:**
- Scalar metrics: $C_{\ell1}$, IoU, NC, F-score.

**Data Structures:**
- Point cloud $S_{rec}$: Set of 3D points sampled from $M_{rec}$.
- Point cloud $S_{gt}$: Set of 3D points sampled from $M_{gt}$ (or voxel grid centers).
- Voxel grid $V_{rec}$: Binary occupancy grid for $M_{rec}$.
- Voxel grid $V_{gt}$: Binary occupancy grid for $M_{gt}$.

**Parameters:**
- Max edge length for subdivision: $L_{max} = 1.5$ cm [S1].
- Sampling density: $\rho = 1$ point/cm² [S1].
- Voxel edge length: $L_{vox} = 5$ cm [S1].
- F-score threshold: $\tau = 5$ cm [S1].

**Steps:**

1. **Mesh Subdivision**: Subdivide all triangles in $M_{rec}$ until every edge length is strictly less than $L_{max} = 1.5$ cm [S1]. This ensures uniform sampling density and accurate visibility detection.

2. **Visibility Detection**:
   - For each vertex $v$ in the subdivided $M_{rec}$, project it into the image plane of each input camera $i$ using $P_i$ and $K_i$.
   - A vertex is **visible** if its projection falls within the image bounds and the depth value at the projection matches the vertex depth within a small tolerance (or if the vertex is not occluded by other geometry in the scene according to the input depth maps) [S1].
   - A triangle is **visible** if at least one of its vertices is visible.
   - **Culling**: Remove all triangles from $M_{rec}$ that have zero visible vertices. These triangles are either outside the viewing frusta or occluded by other geometry [S1].

3. **Point Cloud Sampling**:
   - Uniformly sample points from the remaining (culled) triangles of $M_{rec}$ at density $\rho = 1$ point/cm² to form point cloud $S_{rec}$ [S1].
   - Similarly, sample points from the corresponding visible regions of the ground truth mesh $M_{gt}$ (or the entire $M_{gt}$ if the metric definition implies global comparison, but the text implies evaluation on surfaces visible in streams) to form $S_{gt}$. *Note: The source specifies evaluating metrics on surfaces visible in input streams, implying $S_{gt}$ should also be restricted to visible regions or the comparison is strictly between visible reconstructions and their corresponding visible GT.*

4. **Chamfer Distance ($C_{\ell1}$) and Normal Consistency (NC)**:
   - Compute $C_{\ell1}$ between $S_{rec}$ and $S_{gt}$:
     $$ C_{\ell1}(S_{rec}, S_{gt}) = \frac{1}{|S_{rec}|} \sum_{p \in S_{rec}} \min_{q \in S_{gt}} \|p - q\|_1 + \frac{1}{|S_{gt}|} \sum_{q \in S_{gt}} \min_{p \in S_{rec}} \|p - q\|_1 $$
     *(Note: Source specifies $C_{\ell1}$, which usually implies L1 norm, but Chamfer is often L2. We stick to the source's notation $C_{\ell1}$)* [S1].
   - Compute NC using cosine similarity between the normal at each point in $S_{rec}$ and the nearest point's normal in $S_{gt}$:
     $$ NC = \frac{1}{|S_{rec}|} \sum_{p \in S_{rec}} \max(0, n_p \cdot n_{\hat{q}}) $$
     where $n_p$ is the normal at $p$, and $n_{\hat{q}}$ is the normal at the nearest neighbor $\hat{q} \in S_{gt}$ [S1].

5. **IoU and F-score Computation**:
   - Voxelize $M_{rec}$ and $M_{gt}$ into binary grids with edge length $L_{vox} = 5$ cm [S1]. Let $V_{rec}$ and $V_{gt}$ be the resulting voxel sets.
   - Compute Intersection-over-Union (IoU):
     $$ \text{IoU} = \frac{|V_{rec} \cap V_{gt}|}{|V_{rec} \cup V_{gt}|} $$
   - Compute F-score with threshold $\tau = 5$ cm:
     - Calculate precision: fraction of voxels in $V_{rec}$ that have a neighbor in $V_{gt}$ within distance $\tau$.
     - Calculate recall: fraction of voxels in $V_{gt}$ that have a neighbor in $V_{rec}$ within distance $\tau$.
     - F-score is the harmonic mean of precision and recall [S1].

**Pitfalls:**
- **Occlusion Handling**: Failing to cull occluded triangles inflates error metrics because the network cannot predict geometry it never saw, and GT may contain hidden surfaces [S1].
- **Visibility Definition**: Ensure the visibility check uses the ground truth camera trajectory and depth data accurately to distinguish between "unseen" and "occluded" if necessary, though the pipeline culls both [S1].

## Subtopics

## Operating characteristics
- **Time Complexity**: Dominated by visibility testing ($O(V \cdot N)$ where $V$ is vertices, $N$ is frames) and nearest-neighbor searches for metrics ($O(|S_{rec}| \log |S_{gt}|)$). Subdivision adds overhead proportional to mesh complexity.
- **Memory Complexity**: Linear in the number of vertices and sampled points. Voxelization requires memory proportional to the scene volume divided by $L_{vox}^3$.
- **Strengths**: Provides a fair evaluation of reconstruction quality by ignoring regions the model could not possibly have reconstructed correctly (occlusions/unseen areas) [S1].
- **Limitations**: Metrics are sensitive to the visibility detection algorithm. If visibility is incorrectly determined, valid geometry may be culled or invalid geometry retained. The choice of $L_{max}$ and sampling density affects precision.
- **When to use**: Evaluating RGB-D reconstruction methods where input views are limited and occlusions are prevalent [S1].
- **When not to use**: When evaluating scene completion or methods that explicitly predict hidden geometry.

## Background
The pipeline was introduced in the context of Neural RGB-D Surface Reconstruction [S1]. It addresses the limitation of standard mesh metrics by acknowledging that reconstruction methods are typically trained only on visible geometry. By restricting evaluation to the viewing frusta of input images, it provides a more meaningful assessment of the method's ability to reconstruct observed surfaces [S1].

## Sources
[S1] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
