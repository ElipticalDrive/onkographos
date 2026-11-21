# Projective TSDF Fusion

Also called: Depth-based TSDF Integration, TSDF Fusion from Depth Images, TSDF Volumetric Fusion

## Overview
Projective TSDF Fusion computes signed distance values for voxels from calibrated depth images by projecting voxel centers into the camera frame and comparing projected depth with observed depth [S1]. It differs from explicit surface conversion methods by operating directly on depth buffers rather than requiring a pre-fused mesh or point cloud, enabling incremental integration of raw sensor data [S1].

## How it works
**Inputs:** A global voxel grid, a sequence of depth images $D(u,v)$, camera intrinsics $\mathbf{K} = \{f_x, f_y, c_x, c_y\}$, and camera extrinsics $(R, \mathbf{t})$ [S1]. A truncation threshold $\tau$ defines the valid distance range [S1].

**Outputs:** A global Truncated Signed Distance Field (TSDF) volume where each voxel stores a fused TSDF value and a weight [S1].

**Data Structures:** The global TSDF volume stores per-voxel distance values and accumulation weights [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def projective_tsdf_fusion(voxel_grid, depth_images, intrinsics, extrinsics, tau):
    # Initialize TSDF values and weights to zero [G]
    tsdf_values = zeros_like(voxel_grid)
    weights = zeros_like(voxel_grid)
    
    for D, (R, t) in zip(depth_images, extrinsics):
        for p in voxel_grid:
            # Transform voxel center to camera frame [S1]
            p_c = R.T @ (p - t)
            x, y, z = p_c
            
            # Ensure positive depth to avoid division by zero [G]
            if z <= 0:
                continue
                
            # Project to 2D image coordinates [S1]
            u = intrinsics.fx * (x / z) + intrinsics.cx
            v = intrinsics.fy * (y / z) + intrinsics.cy
            
            # Validate pixel bounds and depth validity [G]
            if not (0 <= u < W and 0 <= v < H):
                continue
            z_hat = D[v, u]
            if z_hat is invalid:
                continue
                
            # Compute projective signed distance [S1]
            s = z_hat - z
            
            # Apply truncation threshold [S1]
            if abs(s) > tau:
                continue
                
            # Fuse measurement via weighted averaging [S1]
            w = 1.0  # [G]
            weights[p] += w
            tsdf_values[p] = (weights[p] - w) * tsdf_values[p] / weights[p] + w * s / weights[p]
            
    return tsdf_values
```

**Parameters:**
*   $\tau$ (Truncation threshold): Not given in the sources for the general method, but defines the valid distance range [S1].
*   $f_x, f_y, c_x, c_y$: Camera intrinsics [S1].
*   $R, \mathbf{t}$: Camera extrinsics [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point errors in $z$ calculation can affect depth comparison [G].
*   **Boundaries:** Voxels outside the image plane or with $z \le 0$ must be handled to avoid division by zero or invalid memory access [G].
*   **Degenerate Input:** Invalid depth values (e.g., 0 or infinity) must be filtered out before computing $s(\mathbf{p})$ [G].

## Subtopics

### [Single Layer TSDF Update](13744-single-layer-tsdf-update.md)
Single Layer TSDF Update is the core integration step in Projective TSDF Fusion that computes a signed distance field (SDF) sample for a specific voxel from a new depth measurement and merges it into the global map using a weighted average. It differs from static TSDF integration by supporting dynamic scale allocation within a multi-resolution octree, ensuring that distant voxels are not allocated as contiguous blocks and that the voxel size remains proportional to the camera distance.

### TSDF Sub-volume Sampling and Preprocessing
This variant involves preprocessing TSDF data for machine learning applications [S3]. It fuses sequences of depth images into a TSDF using truncation limits $d_{min} = -0.04$ and $d_{max} = 0.1$ [S3]. Sub-volumes of size $16 \times 16 \times 16$ voxels are sampled from the TSDF at every eight voxels along each spatial dimension [S3]. The indexing order along each axis is permuted for every sample to generate five additional reflections at each location [S3]. Distance values are mapped from the interval $[d_{min}, d_{max}]$ to $[0, 1]$ [S3]. Sub-volumes where the mean sum of normalized distances (in $[0, 1]$) is below 0.85 are discarded to avoid disproportionate empty space samples, and a small proportion of empty samples is explicitly included to balance the dataset [S3]. The resulting dataset has dimensionality $n = 4096$ ($16^3$) and size $m = 200,000$ samples, structured as $X \in \{\mathbb{R}^{m \times n} \mid 0 \le x_{i,j} \le 1\}$ [S3].

## Operating characteristics
*   **Time Complexity:** $O(V \cdot N)$, where $V$ is the number of voxels in the frustum and $N$ is the number of depth images [G].
*   **Memory Complexity:** $O(V_{total})$, where $V_{total}$ is the total number of voxels in the global volume [G].
*   **Strengths:** Directly uses depth sensor data; robust to noise via weighted averaging; supports incremental updates [G].
*   **Limitations:** Requires accurate camera calibration; performance depends on voxel grid resolution and frustum size; truncation limits must be chosen carefully to balance detail and noise [G].
*   **When to use:** Real-time 3D reconstruction from depth cameras (e.g., KinectFusion) [G].
*   **When not to use:** When only explicit meshes are available without depth buffers (use Explicit Surface to TSDF Conversion instead) [G].

## Background
Projective TSDF fusion was introduced in volumetric fusion systems such as KinectFusion [Newcombe et al. 2011] and voxel hashing systems [Nießner et al. 2013] [S1]. It builds on the weighted averaging fusion technique by Curless and Levoy [Curless and Levoy 1996] [S1]. The method is widely used in real-time 3D reconstruction from depth sensors [S1].

## Sources
[S1] Implicit Surface Compression -- with Good Old Discrete Cosine Transform and Motion Compensation, https://doi.org/10.1145/3811298, fetched 2026-10-30
[S2] PDF Interactive Editing of Voxel-Based Signed Distance Fields, http://wscg.zcu.cz/WSCG2022/journal/B97-full.pdf, fetched 2026-10-30
[S3] Compressed Voxel-Based Mapping Using Unsupervised Learning, https://doi.org/10.3390/robotics6030015, fetched 2026-10-26
