# Sparse View Selection and Dataset Configuration

Also called: Dataset Configuration, Sparse View Selection

## Overview
Sparse View Selection and Dataset Configuration defines the protocol for selecting a minimal subset of input images and preparing the corresponding ground truth geometry for training and evaluating novel view synthesis and geometric reconstruction models. It addresses the challenge of training neural representations when full multi-view coverage is computationally prohibitive or unavailable. It is distinguished from sibling methods like [Novel View Synthesis Rendering](6402-novel-view-synthesis-rendering.md) by its focus on input data curation rather than rendering, and from [Data Augmentation](20546-data-augmentation.md) by its deterministic or specific stochastic selection of source views rather than transformation of existing ones.

## How it works
**Inputs:**
- A set of source images with known camera poses.
- Ground truth 3D geometry (point clouds or meshes) if available.
- Dataset-specific configuration parameters (number of training/test views, visibility thresholds).

**Outputs:**
- A training dataset consisting of $N_{train}$ selected views.
- A testing dataset consisting of $N_{test}$ selected views.
- A filtered ground truth geometry mask.

**Data Structures:**
- `ViewSet`: A list of objects, each containing image data, camera intrinsic parameters, and extrinsic pose matrices.
- `GeometryMask`: A boolean mask or point cloud subset indicating valid 3D points.

**Steps:**
1. **Dataset Selection**: Choose the target dataset (e.g., DTU, Tanks & Temples, Mip-NeRF 360, GigaNVS) based on the evaluation goal (geometric reconstruction vs. novel view synthesis) [S1].
2. **View Sampling**:
   - For DTU: Select exactly 4 views for both training and testing [S1].
   - For Tanks & Temples and Mip-NeRF 360: Select 6 views for training and 6 neighboring views for testing [S1].
   - For GigaNVS: Utilize 6 images per scene [S1].
3. **Geometry Filtering**:
   - Sample the ground truth 3D geometry.
   - Calculate visibility for each 3D point from all available source views.
   - Preserve only points visible from at least 2 views [S1].
4. **Noise/Background Removal**:
   - For Tanks & Temples testing, explicitly exclude irrelevant point clouds such as sky regions and background vegetation to improve result quality [S1].
5. **Evaluation Preparation**:
   - Pair the selected views with the filtered geometry for metric calculation (PSNR, SSIM, LPIPS for images; surface metrics for geometry) [S1].

**Parameters:**
- $N_{train}$: Number of training views. Typical values: 4 (DTU), 6 (Tanks & Temples, Mip-NeRF 360, GigaNVS) [S1].
- $N_{test}$: Number of testing views. Typical values: 4 (DTU), 6 (Tanks & Temples, Mip-NeRF 360) [S1].
- $V_{min}$: Minimum visibility count for geometry retention. Value: 2 [S1].

**Edge Cases:**
- **Occlusion**: Points visible from only one view are discarded, potentially creating holes in the ground truth geometry if the sparse views are insufficiently overlapping [S1].
- **Background Artifacts**: Failure to exclude sky/vegetation in datasets like Tanks & Temples can degrade geometric evaluation metrics [S1].

## Subtopics

### [Light Cone Construction](19580-light-cone-construction.md)
Light Cone Construction is a method for generating synthetic astronomical observations (imaging or spectroscopy) from cosmological simulation data by stacking discrete redshift slices into a continuous line-of-sight volume.

### [Random Multi-view Sampling for Training](19415-random-multi-view-sampling-for-training.md)
Random multi-view sampling for training is a data selection strategy used in inverse rendering and novel view synthesis to construct minibatches from a sparse set of input images.

## Operating characteristics
**Time and Memory Complexity:**
- The selection process is $O(N)$ with respect to the number of source views for sampling.
- Geometry filtering is $O(P \cdot V)$ where $P$ is the number of points and $V$ is the number of views, requiring visibility checks [S1].

**Strengths and Limitations:**
- **Strengths**: Reduces computational cost for training and evaluation; standardizes benchmarks across different algorithms [S1].
- **Limitations**: Sparse views (e.g., 4-6 views) present a significant challenge for reconstruction, potentially leading to ambiguous geometry or poor novel view synthesis if the algorithm lacks strong priors [S1].

**When to Use:**
- Use when evaluating geometric reconstruction performance or novel view synthesis under resource constraints [S1].
- Use when comparing algorithms on standard benchmarks (DTU, Tanks & Temples) [S1].

**When Not to Use:**
- Do not use when full multi-view coverage is available and computational resources permit, as sparse configurations artificially limit performance [S1].

## Background
This configuration is primarily used to evaluate geometric reconstruction performance and novel view synthesis capabilities in inverse rendering and neural rendering research [S1]. The specific configurations for DTU (4 views) and Tanks & Temples/Mip-NeRF 360 (6 views) have become standard benchmarks for assessing the effectiveness of algorithms like GBR (Generative Bundle Refinement) in handling sparse input data [S1].

## Sources
[S1] GBR: Generative Bundle Refinement for High-Fidelity Gaussian Splatting With Enhanced Mesh Reconstruction, https://doi.org/10.1109/tcsvt.2025.3596613, fetched 2026-10-29
