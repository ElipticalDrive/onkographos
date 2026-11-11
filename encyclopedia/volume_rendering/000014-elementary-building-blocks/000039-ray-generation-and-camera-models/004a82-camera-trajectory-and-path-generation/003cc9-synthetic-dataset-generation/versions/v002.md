# Synthetic Dataset Generation


## Overview
Synthetic dataset generation creates training data for volume rendering and 3D reconstruction models by simulating sensor observations in computer-generated environments. It solves the problem of lacking real-world labeled data by providing ground-truth geometry, semantic labels, and controlled noise characteristics [S1, S4]. Unlike methods that generate camera trajectories for active planning [S3, unofficial] or reconstruct scenes from sparse inputs [S2], this method focuses on the pipeline of rendering, augmenting, and cleaning synthetic data to match the statistical properties of real sensors. It differs from [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md) by prioritizing data quality and label correctness over trajectory coverage, and from [Novel View Synthesis Rendering](6402-novel-view-synthesis-rendering.md) by providing explicit geometric and semantic ground truth rather than just photometric consistency.

## How it works
The method consists of three main phases: trajectory definition and rendering, data augmentation for model training, and dataset curation/cleaning.

### Inputs and Outputs
**Inputs:**
- 3D scene geometry (meshes, textures).
- Sensor parameters (intrinsics, extrinsics, noise models).
- Control points for trajectory definition.
- Semantic label maps (optional, for supervised learning).

**Outputs:**
- Rendered color images.
- Rendered depth maps (with added noise).
- Semantic label maps.
- Camera poses (trajectory).
- A structured dataset $\mathcal{D}$ suitable for training neural fields (e.g., SDDF, NeRF).

### Data Structures
1. **Ray Measurement**: A tuple $(\mathbf{v}_i, r_{t,i})$ where $\mathbf{v}_i$ is the ray direction in sensor frame and $r_{t,i}$ is the range measurement [S2].
2. **Dataset Sample**: A tuple $(\mathbf{p}_j, \mathbf{v}_j, f^*_j, i^*_j, s^*_j)$ where:
   - $\mathbf{p}_j \in \mathbb{R}^3$: Ray origin (sensor position) [S2].
   - $\mathbf{v}_j \in S^2$: Ray direction in world frame [S2].
   - $f^*_j$: SDDF measurement (signed distance) [S2].
   - $i^*_j \in \{-1, 1\}$: Intersection indicator [S2].
   - $s^*_j \in \{-1, 1\}$: Sign indicator [S2].
3. **View Space Representation**:
   - **Hemisphere**: Defined by maximum radius $R_{max}$ and center. Viewpoint represented by spherical coordinates $(\theta, \phi, r)$ constrained to point toward the object center [S3, unofficial].
   - **3D Bounding Box**: Defined by corners derived from raw training data. Viewpoint represented by 6DoF $SE(3)$ transformation [S3, unofficial].

### Steps

#### Phase 1: Trajectory Definition and Rendering
1. **Define Trajectory**: Select control points manually. Interpolate camera poses using a Catmull-Rom spline [S1].
2. **Render Images**: For each pose in the trajectory, render color and depth images using a renderer (e.g., BlenderProc) [S1].
3. **Apply Noise**: Add noise to depth maps to simulate real sensor noise (e.g., Gaussian, quantization) [S1].
   - *Note*: For specific datasets like ICL, use provided noisy depth/color data instead of rendering [S1].
4. **Filter Invalid Views**: Filter out images where the camera is inside geometry (e.g., furniture) [S4].

#### Phase 2: Dataset Augmentation (SDDF Specific)
To train models like SDDF, the dataset must be balanced in sign indicators [S2].

1. **Generate Positive Samples**: For each sensor measurement $(\mathbf{v}_i, r_{t,i})$ at pose $T_t = (\mathbf{p}_t, R_t)$:
   - Set origin: $\mathbf{p}_j = \mathbf{p}_t$
   - Set direction: $\mathbf{v}_j = R_t \mathbf{v}_i$
   - Set SDDF value: $f^*_j = r_{t,i}$
   - Set intersection indicator: $i^*_j = 1$
   - Set sign indicator: $s^*_j = 1$
   [S2]

2. **Generate Negative Samples**: For each positive sample $(\mathbf{p}_j, \mathbf{v}_j, f^*_j, i^*_j, s^*_j)$:
   - Calculate offset point: $\mathbf{p}' = \mathbf{p}_j + (f^*_j + \epsilon) \mathbf{v}_j$, where $\epsilon > 0$ is a small constant [S2].
   - Set origin: $\mathbf{p}'$
   - Set direction: $\mathbf{v}_j$ (same direction)
   - Set SDDF value: $-\epsilon$
   - Set intersection indicator: $1$
   - Set sign indicator: $-1$
   [S2]

3. **Skip Intersection Augmentation**: Do not generate non-intersecting rays ($i^*_j = -1$) as it is inefficient and does not improve reconstruction results [S2].

#### Phase 3: View Space Sampling (Active Viewpoint Generation)
For active mapping or information gain calculation [S3, unofficial]:

1. **Define View Space**:
   - **Synthetic-NeRF**: Hemisphere with $R_{max} = 4$. Viewpoint in spherical coordinates, constrained to point at object center [S3, unofficial].
   - **Tanks & Temple**: 3D bounding box from raw training set. Viewpoint in $SE(3)$ [S3, unofficial].
2. **Sample Waypoints**: Uniformly sample $N_{wp}$ waypoints from a Bézier trajectory [S3, unofficial].
3. **Select Viewpoints**: Select $N_{view}$ viewpoints from waypoints for information gain calculation [S3, unofficial].
4. **Sample 3D Points**: Instead of sampling all pixels, sample 3D points from $1/K$ of the pixels for information gain calculation [S3, unofficial].

#### Phase 4: Dataset Curation and Cleaning
1. **Filter Geometry**: Remove images rendered from inside objects [S4].
2. **Semantic Label Correction**:
   - Apply a segmentation model (e.g., LSeg) to predict semantic labels [S4].
   - Compare predicted labels with ground truth.
   - Ignore points/labels with near-zero accuracy [S4].
   - Manually merge ambiguous labels (e.g., 'rug' and 'floor') [S4].

### Parameters
- **Trajectory Interpolation**: Catmull-Rom spline [S1].
- **Renderer**: BlenderProc [S1].
- **Noise Model**: Real sensor noise simulation [S1].
- **SDDF Offset**: $\epsilon > 0$ (small constant) [S2].
- **View Space Radius**: $R_{max} = 4$ (Synthetic-NeRF) [S3, unofficial].
- **Waypoint Count**: $N_{wp} = 100$ [S3, unofficial].
- **Viewpoint Count**: $N_{view} = 10$ [S3, unofficial].
- **Pixel Sampling Ratio**: $1/100$ [S3, unofficial].
- **Clipping Planes**: Near/Far $(0.1, 10.0)$, except office_3 $(0.1, 15.0)$ [S4].
- **Voxelization Edge Length**: $5$ cm (for IoU evaluation) [S1].
- **F-score Threshold**: $5$ cm [S1].

### Edge Cases and Pitfalls
- **Unbalanced Signs**: Initial dataset has $s^*_j = 1$ and $f^*_j \ge 0$. Negative samples are required for SDDF training [S2].
- **Inefficient Augmentation**: Generating non-intersecting rays ($i^*_j = -1$) is inefficient and unnecessary [S2].
- **Invalid Views**: Cameras inside furniture produce unrealistic data; must be filtered [S4].
- **Label Ambiguity**: Synthetic datasets may have indistinguishable semantic classes (e.g., rug vs. floor); manual merging or ignoring is required [S4].
- **Occlusion Culling**: Vertices occluded or outside frusta must be culled before error evaluation to avoid unpredictable network outputs [S1].

## Subtopics

### Dataset Generation and Augmentation
A procedure to convert sensor poses and range measurements into a dataset $\mathcal{D}$ suitable for training SDDF models, specifically addressing sign imbalance. It generates positive samples from direct measurements and negative samples by extending rays slightly behind the surface. It explicitly avoids generating non-intersecting rays due to inefficiency and lack of benefit [S2].

### Synthetic-NeRF View Space and Sampling
Defines the camera view space (hemisphere or bounding box) and sampling strategy for active viewpoint generation. It uses spherical coordinates for Synthetic-NeRF scenes and $SE(3)$ for Tanks & Temple. It samples waypoints from Bézier trajectories and reduces computational cost by sampling only a fraction of pixels for information gain calculation [S3, unofficial].

### Replica Dataset Preparation
A curation pipeline for the Replica dataset. It filters invalid camera poses (inside furniture), uses LSeg for semantic label prediction, and manually fixes label ambiguities (e.g., merging rug/floor). It uses specific near/far clipping planes depending on the scene [S4].

## Operating characteristics
- **Time Complexity**: Rendering is parallelizable per camera pose [G]. Information gain calculation is reduced by sampling $1/100$ of pixels [S3, unofficial].
- **Memory Complexity**: Depends on scene resolution and number of samples. SDDF augmentation doubles the dataset size (positive + negative samples) [S2].
- **Performance**: Synthetic data allows for fair comparison with initial pose estimation methods like BundleFusion [S1].
- **Strengths**: Provides ground truth geometry and semantics; controllable noise; reproducible.
- **Limitations**: Domain gap between synthetic and real data; requires manual curation for semantic labels [S4].
- **When to Use**: When real labeled data is scarce or expensive; for training neural fields requiring explicit geometric supervision (SDDF); for active mapping research [S2][S3, unofficial].
- **When Not to Use**: When domain adaptation is not feasible; when real-world sensor artifacts are critical and cannot be simulated [S1].

## Background
Synthetic dataset generation originated from the need for ground-truth data in 3D reconstruction and neural rendering research. Early works used BlenderProc to render scenes with Catmull-Rom spline trajectories [S1]. Later, methods like SDDF required specific data augmentation to handle signed distance functions [S2]. Active mapping research introduced view space sampling strategies for information gain calculation [S3, unofficial]. Recent works have focused on curating synthetic datasets like Replica to address semantic label ambiguities [S4].

## Sources
[S1] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
[S2] Learning Scene-Level Signed Directional Distance Function With Ellipsoidal Priors and Neural Residuals., https://doi.org/10.1109/tpami.2026.3688658, fetched 2026-10-26
[S3] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S4] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
