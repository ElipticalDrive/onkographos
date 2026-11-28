# Cloud-to-Cloud Comparison

Also called: C2C distance metric, C2C metric comparison, C2C metric evaluation, Cloud-to-Cloud (C2C) distance calculation, Cloud-to-Cloud (C2C) metric comparison, Euclidean Distance Assessment, Point Cloud Registration Error, point cloud distance evaluation

## Overview
Cloud-to-Cloud (C2C) comparison is a quantitative evaluation method that assesses the geometric accuracy and alignment of a reconstructed 3D model against a ground truth (GT) reference by computing the relative Euclidean distances between corresponding 3D sample points [S1, S2]. It differs from global registration metrics by focusing on the per-point deviation magnitude rather than just the transformation error, and it is distinct from surface-based metrics (like Chamfer distance) by typically operating on discrete point samples rather than continuous meshes [S1, S2].

## How it works
**Inputs:**
1.  **Reference Point Cloud ($P_{ref}$):** The ground truth dataset or high-accuracy baseline model (e.g., from photogrammetry or laser scanning) [S1, S2].
2.  **Target Point Cloud ($P_{tgt}$):** The reconstructed dataset to be evaluated (e.g., from NeRF or photogrammetry) [S1, S2].
3.  **Alignment Parameters:** A rigid transformation (rotation $R$ and translation $t$) aligning $P_{tgt}$ to $P_{ref}$, derived from control points or co-registration [S1, S2].

**Outputs:**
Statistical summaries of the distance distribution, typically including:
*   Root Mean Square Error (RMSE)
*   Mean Absolute Error (MAE)
*   Standard Deviation (STD)
*   Maximum Distance [S1, S2]

**Data Structures:**
*   $P_{ref}$: List of $N$ points, each represented as a 3D vector $\mathbf{p}_i \in \mathbb{R}^3$.
*   $P_{tgt}$: List of $M$ points, each represented as a 3D vector $\mathbf{q}_j \in \mathbb{R}^3$.
*   Distance Map: An array of size $N$ storing the nearest neighbor distance for each point in $P_{ref}$.

**Numbered Steps:**

1.  **Preprocessing and Alignment:**
    *   Export point clouds from both the reference and target models.
    *   If the target model contains internal voxels or volume data (common in NeRF outputs), filter the point cloud to retain only points belonging to the domain of the reconstructed object (the shell) to avoid inflated error values [S2].
    *   Co-register $P_{tgt}$ to $P_{ref}$ using control points or an alignment algorithm to ensure they share the same coordinate system [S1, S2].

2.  **Nearest Neighbor Search:**
    *   For each point $\mathbf{p}_i$ in the reference cloud $P_{ref}$, find the nearest neighbor $\mathbf{q}_{k}$ in the target cloud $P_{tgt}$.
    *   This is typically implemented using a spatial index (e.g., KD-Tree) for efficiency [S2].

3.  **Distance Calculation:**
    *   Compute the Euclidean distance $d_i$ for each point $\mathbf{p}_i$:
        $$ d_i = \| \mathbf{p}_i - \mathbf{q}_{k} \|_2 = \sqrt{(p_{i,x} - q_{k,x})^2 + (p_{i,y} - q_{k,y})^2 + (p_{i,z} - q_{k,z})^2} $$
    *   Note: Some implementations compute distances bidirectionally (from ref to tgt and tgt to ref), but the primary C2C metric described in the sources computes the distance from the reference points to their nearest neighbors in the target cloud [S2].

4.  **Statistical Aggregation:**
    *   Calculate the Mean Absolute Error (MAE):
        $$ \text{MAE} = \frac{1}{N} \sum_{i=1}^{N} d_i $$
    *   Calculate the Root Mean Square Error (RMSE):
        $$ \text{RMSE} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} d_i^2} $$
    *   Calculate the Standard Deviation (STD) of the distances:
        $$ \text{STD} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (d_i - \text{MAE})^2} $$
    *   Identify the Maximum Distance:
        $$ d_{\max} = \max_{i} (d_i) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_c2c_metrics(reference_cloud, target_cloud):
    """
    Computes C2C metrics between a reference point cloud and a target point cloud.
    
    Args:
        reference_cloud: List of numpy arrays or Nx3 array, shape (N, 3).
        target_cloud: List of numpy arrays or Mx3 array, shape (M, 3).
        
    Returns:
        dict: Dictionary containing RMSE, MAE, STD, and Max Distance.
    """
    # Convert to numpy arrays if necessary
    P_ref = np.array(reference_cloud)
    P_tgt = np.array(target_cloud)
    
    N = P_ref.shape[0]
    distances = np.zeros(N)
    
    # Step 1: Build KD-Tree for efficient nearest neighbor search in target cloud
    # [G] Using scipy.spatial.cKDTree for implementation clarity
    from scipy.spatial import cKDTree
    tree = cKDTree(P_tgt)
    
    # Step 2 & 3: Compute Euclidean distance for each reference point to its nearest neighbor
    # query returns (distances, indices)
    dists, _ = tree.query(P_ref)
    distances = dists
    
    # Step 4: Aggregate statistics
    if N == 0:
        return {"RMSE": 0.0, "MAE": 0.0, "STD": 0.0, "Max": 0.0}
        
    mae = np.mean(distances)
    rmse = math.sqrt(np.mean(distances**2))
    std = math.sqrt(np.mean((distances - mae)**2))
    max_dist = np.max(distances)
    
    return {
        "RMSE": rmse,
        "MAE": mae,
        "STD": std,
        "Max": max_dist
    }
```

**Parameters:**
*   **Image Count:** The number of input images used to generate the target cloud affects accuracy. For example, in industrial datasets, 140 images were used for both photogrammetry and NeRF, whereas for synthetic datasets, photogrammetry used 25 images while NeRF used 200 [S1].
*   **Unit of Measurement:** Distances are typically reported in millimeters (mm) or meters (m) depending on the scale of the object [S1, S2].

**Edge Cases and Pitfalls:**
*   **Internal Voxels:** Comparing full point clouds that include internal voxels (common in NeRF outputs) yields artificially high error values. Only points on the object's shell should be considered for accurate metric comparison [S2].
*   **Surface Properties:** Metallic, shiny, or texture-less surfaces cause significant errors in reconstruction methods like NeRF, leading to higher C2C distances [S1].
*   **Image Count Diminishing Returns:** For photogrammetry, adding more images beyond a certain baseline does not improve accuracy, whereas NeRF may require all available images to avoid degradation [S1].

## Operating characteristics
*   **Time Complexity:** The dominant operation is the nearest neighbor search. With a KD-Tree, the query time is approximately $O(N \log M)$, where $N$ is the number of reference points and $M$ is the number of target points. Without spatial indexing, it is $O(N \times M)$ [S2].
*   **Memory Complexity:** $O(N + M)$ to store the point clouds and the distance array [S2].
*   **Measured Performance:**
    *   **Industrial Object:** Photogrammetry RMSE = 0.85 mm, NeRF RMSE = 0.78 mm [S1].
    *   **Synthetic Object:** Photogrammetry RMSE = 0.13 mm, NeRF RMSE = 0.19 mm [S1].
    *   **Ignatius (Statue):** Photogrammetry RMSE = 2.50 mm, NeRF RMSE = 5.50 mm [S1].
    *   **Truck:** Photogrammetry RMSE = 16.0 mm, NeRF RMSE = 38.0 mm [S1].
    *   **Buddha Dataset Profile:** Max distance = 0.07 cm, Mean = 0.002 m, STD = 0.002 m [S2].
*   **Strengths:** Provides an intuitive, per-point error metric that is easy to interpret and visualize (e.g., color-coded distance maps) [S1, S2].
*   **Limitations:** Sensitive to noise and outliers; requires accurate alignment; does not account for surface topology directly (only point proximity) [S1, S2].

## Background
The Cloud-to-Cloud comparison method is widely used in 3D reconstruction evaluation, particularly in comparing traditional photogrammetry (e.g., Colmap, Agisoft Metashape) with Neural Radiance Fields (NeRF) based methods (e.g., Instant-NGP) [S1, S2]. It is implemented in standard 3D processing software such as Cloud Compare [S2]. The metric has been applied to various object types, including industrial parts, synthetic models, and cultural heritage artifacts (statues) [S1, S2].

## Sources
[S1] A Critical Analysis of NeRF-Based 3D Reconstruction, https://doi.org/10.3390/rs15143585, fetched 2026-10-26
[S2] Assessment of 3D Model for Photogrammetric Purposes Using AI Tools Based on NeRF Algorithm, https://doi.org/10.3390/heritage6080301, fetched 2026-10-28
