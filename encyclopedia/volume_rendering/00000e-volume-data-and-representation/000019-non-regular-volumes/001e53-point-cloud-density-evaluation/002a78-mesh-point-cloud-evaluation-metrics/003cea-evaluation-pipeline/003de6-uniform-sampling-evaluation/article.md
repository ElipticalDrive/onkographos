# Uniform Sampling Evaluation

Also called: Uniform Mesh Sampling for Evaluation

## Overview
Uniform Sampling Evaluation is a geometric evaluation protocol for surface reconstruction that eliminates vertex-density bias by uniformly sampling a fixed number of points from the reconstructed mesh surface within the ground truth (GT) crop volume [S1]. It ensures a strictly fair geometric comparison by decoupling the evaluation density from the mesh tessellation quality [S1]. Unlike legacy vertex-based metrics, which construct evaluation point clouds from mesh vertices and face centers, this method prevents scores from being artificially inflated by denser tessellations [S1]. It is primarily used for evaluating Gaussian-based surface reconstruction on the Tanks and Temples (T&T) and structured scan datasets [S1].

## How it works
**Inputs:**
1.  Reconstructed mesh $M$ (typically extracted from Gaussian splatting or similar representations).
2.  Ground truth crop volume $V_{GT}$ (defined by the acquisition bounds of the GT laser scans) [S1].
3.  Target number of evaluation points $N$ [S1].

**Outputs:**
1.  A set of $N$ uniformly sampled points $P_{pred}$ on the surface of $M$ restricted to $V_{GT}$ [S1].
2.  Geometric metrics (e.g., F1-score, Chamfer Distance) computed between $P_{pred}$ and the GT point cloud $P_{GT}$ [S1].

**Data Structures:**
*   $P_{pred}$: A list or array of $N$ 3D coordinates $(x, y, z)$.
*   $P_{GT}$: The ground truth point cloud, typically derived from laser scans, filtered to the same crop volume $V_{GT}$ [S1].

**Steps:**
1.  **Define Crop Volume:** Identify the bounding volume $V_{GT}$ corresponding to the ground truth acquisition region [S1].
2.  **Uniform Sampling:** Generate $N$ points uniformly distributed on the surface of the reconstructed mesh $M$, considering only the portion of the mesh intersecting $V_{GT}$ [S1].
    *   *Note:* The exact algorithm for uniform mesh sampling (e.g., stratified sampling based on face area) is not specified in the source [S1]. [G] A standard implementation involves calculating the total surface area of the mesh within $V_{GT}$, assigning each face a probability proportional to its area, and sampling points within the selected faces using barycentric coordinates.
3.  **Metric Computation:** Compute geometric metrics between the sampled points $P_{pred}$ and the GT point cloud $P_{GT}$ [S1]. Common metrics include the F1-score (based on Chamfer Distance thresholds) [S1].

**Parameters:**
*   $N$ (Fixed number of sampled points): Controls the density of the evaluation point cloud. The specific value is not given in the sources [S1].

**Pitfalls:**
*   **Complex/Sparse Geometry:** Uniform sampling may unfairly penalize performance on scenes with complex or sparse geometry (e.g., Courthouse, Meetingroom) [S1].
*   **Incomplete Meshes:** Uniform sampling alone does not account for incomplete meshes that might mimic GT bias (i.e., missing parts that happen to align with GT data) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniform_sampling_evaluation(mesh, gt_crop_volume, gt_point_cloud, num_samples):
    """
    Evaluates a reconstructed mesh against ground truth using uniform sampling.
    
    Args:
        mesh: The reconstructed surface mesh.
        gt_crop_volume: Bounding volume defining the region of interest.
        gt_point_cloud: Ground truth point cloud.
        num_samples: Fixed number of points to sample from the mesh.
        
    Returns:
        dict: Computed geometric metrics (e.g., F1-score).
    """
    # 1. Filter mesh to GT crop volume
    # [G] Implementation detail: Clip or cull mesh triangles that do not intersect 
    # gt_crop_volume. Let the resulting mesh be mesh_cropped.
    mesh_cropped = clip_mesh_to_volume(mesh, gt_crop_volume)
    
    # 2. Uniformly sample points from the cropped mesh
    # [G] Implementation detail: Calculate total surface area of mesh_cropped.
    # Sample faces with probability proportional to area.
    # For each selected face, sample a point using barycentric coordinates.
    # Repeat until num_samples points are collected.
    sampled_points = uniform_sample_mesh(mesh_cropped, num_samples)
    
    # 3. Compute metrics against GT
    # [G] Implementation detail: Compute Chamfer Distance between sampled_points 
    # and gt_point_cloud. Convert to F1-score using a standard threshold (e.g., 0.01m).
    metrics = compute_chamfer_and_f1(sampled_points, gt_point_cloud)
    
    return metrics
```

## Subtopics

### Virtual Scanning Evaluation
Virtual Scanning Evaluation is an evaluation protocol that simulates the original laser scan data acquisition process to account for incomplete meshes and acquisition bias, complementing Uniform Sampling [S1]. It addresses the limitation of Uniform Sampling alone, which fails to account for incomplete meshes that mimic GT bias [S1]. By rendering depth maps from input camera poses and back-projecting them, it reveals true reconstruction quality that uniform sampling might miss [S1]. It is used alongside Uniform Sampling for robust evaluation on T&T and Mip-NeRF 360 datasets [S1].

## Operating characteristics
*   **Bias Elimination:** Eliminates the bias inherent in legacy vertex-based metrics, which are highly sensitive to tessellation density (e.g., extracting meshes with 9 pivots instead of 2 artificially inflates scores) [S1].
*   **Performance:** On the Tanks and Temples dataset, this protocol sets a new state-of-the-art for unbiased evaluation among full-scene methods [S1].
*   **Limitations:** It may be suboptimal for scenes with complex or sparse geometry, potentially penalizing methods unfairly in those specific cases [S1]. It also does not inherently address incomplete meshes that align with GT data, necessitating the use of Virtual Scanning Evaluation as a complement [S1].
*   **Hardware/Resolution:** Performance metrics (e.g., F1 scores) are reported for specific datasets (T&T, DTU) but specific hardware or resolution scaling characteristics are not detailed in the sources [S1].

## Background
The method was proposed in "From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians" [S1]. It was introduced to address fundamental flaws in the standard evaluation practice on Tanks and Temples (T&T) and structured scan datasets [S1]. The conventional approach was found to be flawed because it built predicted surface point clouds from mesh vertices and face centers, biasing metrics toward denser tessellations, and because ground data acquisition (relying on laser scans) unfairly penalized methods that correctly reconstructed occluded geometry and surfaces visible at grazing angles [S1]. The authors propose Uniform Sampling Evaluation and Virtual Scanning Evaluation as robust alternatives [S1].

## Sources
[S1] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
