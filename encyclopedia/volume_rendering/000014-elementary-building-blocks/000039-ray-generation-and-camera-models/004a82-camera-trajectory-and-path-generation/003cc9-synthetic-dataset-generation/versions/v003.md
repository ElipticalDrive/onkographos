# Synthetic Dataset Generation


## Overview
Synthetic dataset generation constructs training data for neural rendering and 3D reconstruction models by procedurally defining camera trajectories, rendering ground-truth observations, and applying data augmentation to balance statistical properties. It solves the problem of lacking paired, high-quality sensor data for training by simulating the acquisition process in a controlled environment. Unlike [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md) which focuses on specific geometric constraints for elongated objects, or [Sparse Trajectory Generation](7215-sparse-trajectory-generation.md) which limits view counts, this method emphasizes the full pipeline from trajectory definition to dataset formatting, including noise injection and sign-balancing augmentation. It builds on [Camera Trajectory and Path Generation](6410-camera-trajectory-and-path-generation.md) by providing the concrete steps to convert those trajectories into usable training samples.

## How it works
The method varies significantly depending on the target model (e.g., surface reconstruction vs. active mapping). The following sections detail the two primary implementations found in the literature: general neural surface reconstruction datasets and Signed Directional Distance Function (SDDF) datasets.

### General Neural Surface Reconstruction (e.g., Neural RGB-D)
This approach generates color and depth images from synthetic scenes to train networks that reconstruct surfaces from RGB-D inputs.

**Inputs:**
- A set of manually selected 3D control points defining the camera path.
- A synthetic 3D scene mesh (e.g., from BlendSwap or ICL dataset).
- A rendering engine (e.g., BlenderProc).

**Outputs:**
- A dataset $D$ containing pairs of color images, depth maps, and camera poses.

**Data Structures:**
- **Camera Pose:** $4 \times 4$ transformation matrix $T \in \mathbb{R}^{4 \times 4}$ mapping world coordinates to camera coordinates.
- **Depth Map:** 2D array $Z \in \mathbb{R}^{H \times W}$, where each element represents the distance from the camera center to the intersected surface.
- **Color Image:** 2D array $C \in \mathbb{R}^{H \times W \times 3}$, normalized to $[0, 1]$ or $[0, 255]$.

**Steps:**
1. **Define Trajectory:** Construct a continuous camera path using Catmull-Rom spline interpolation through the manually selected control points [S1].
   - Let control points be $P_0, P_1, \dots, P_n$.
   - The position $C(t)$ for $t \in [0, 1]$ is interpolated between $P_i$ and $P_{i+1}$.
2. **Sample Poses:** Discretize the spline into $N$ camera poses. For each pose, compute the orientation such that the camera looks at the scene center or along the tangent of the spline.
3. **Render Observations:** For each pose $T_k$:
   - Render the depth map $Z_k$ using the synthetic scene geometry.
   - Render the color image $C_k$ using the scene materials and lighting.
   - Store the pose $T_k$, depth $Z_k$, and color $C_k$ [S1].
4. **Inject Noise:** Apply sensor noise to the depth maps to simulate real-world depth sensor inaccuracies.
   - Let $Z_{clean}$ be the rendered depth.
   - Generate noisy depth $Z_{noisy} = Z_{clean} + \epsilon$, where $\epsilon$ is drawn from a noise distribution (e.g., Gaussian or uniform) characterized by real sensor specifications [S1].
   - *Note:* For specific scenes like ICL, pre-provided noisy depth/color data may be used instead of rendering [S1].
5. **Culling and Evaluation Prep:** For evaluation, cull vertices not visible in any training view or occluded by geometry. Sample points from the remaining geometry at a density of 1 point per cm² for error metric computation (e.g., F-score with 5 cm threshold) [S1].

**Parameters:**
- **Spline Type:** Catmull-Rom [S1].
- **Control Points:** Manually selected, number varies by scene complexity.
- **Noise Model:** Simulates real depth sensor noise [S1].
- **Evaluation Threshold:** 5 cm for voxelization and F-score [S1].

**Pitfalls:**
- **Occlusion:** Rendering depth from a single viewpoint may miss surfaces occluded by foreground geometry. Training data must cover the object from multiple angles to ensure full reconstruction [S1].
- **Coordinate Systems:** Ensure consistent coordinate systems between the synthetic scene, the camera model, and the rendering engine.

### SDDF Dataset Generation and Augmentation
This approach generates datasets specifically for training Signed Directional Distance Function (SDDF) models, requiring balanced sign indicators and directional information.

**Inputs:**
- Sensor poses $T_t$ and range measurements $Z_t = \{v_i, r_{t,i}\}_{i=1}^N$, where $v_i$ is the ray direction in sensor frame and $r_{t,i}$ is the range measurement.

**Outputs:**
- A dataset $D = \{p_j, v_j, f^*_j, i^*_j, s^*_j\}_j$ suitable for training.
  - $p_j \in \mathbb{R}^3$: Ray origin.
  - $v_j \in S^2$: Ray direction.
  - $f^*_j$: SDDF measurement.
  - $i^*_j \in \{-1, 1\}$: Intersection indicator.
  - $s^*_j \in \{-1, 1\}$: Sign indicator.

**Data Structures:**
- **Sample Tuple:** $(p_j, v_j, f^*_j, i^*_j, s^*_j)$.
- **Ray Direction:** Normalized vector $v_j$.

**Steps:**
1. **Convert Raw Measurements to Samples:** For each ray measurement $(v_i, r_{t,i})$ from sensor position $p_t$ and orientation $R_t$:
   - Set origin: $p_j = p_t$.
   - Transform direction to world frame: $v_j = R_t v_i$.
   - Set SDDF measurement: $f^*_j = r_{t,i}$.
   - Set intersection indicator: $i^*_j = 1$ (assuming the ray intersects the surface).
   - Set sign indicator: $s^*_j = 1$ (assuming the ray points outward or towards the surface from the outside).
   - Add $(p_j, v_j, f^*_j, i^*_j, s^*_j)$ to dataset $D$ [S2].
2. **Augment with Negative Samples:** The initial dataset is unbalanced ($s^*_j = 1$ and $f^*_j \ge 0$ for all samples). Generate negative samples to balance the sign indicator.
   - For each existing sample $(p_j, v_j, f^*_j, i^*_j, s^*_j)$:
     - Calculate a point slightly behind the observed surface: $p' = p_j + (f^*_j + \epsilon) v_j$, where $\epsilon > 0$ is a small offset.
     - Create negative sample: $(p', v_j, -\epsilon, 1, -1)$.
     - Add this negative sample to $D$ [S2].
3. **Intersection Indicator Handling:** Do not augment with non-intersecting rays ($i^*_j = -1$). Generating such rays is inefficient and experiments show it does not improve reconstruction results [S2].

**Parameters:**
- **Offset $\epsilon$:** Small positive value to place the negative sample behind the surface. Value not explicitly given in sources [S2].

**Pitfalls:**
- **Sign Imbalance:** Failing to augment with negative samples results in a model that cannot predict negative SDDF values, limiting its ability to represent the interior or outside of the object correctly [S2].
- **Inefficient Augmentation:** Attempting to generate non-intersecting rays for $i^*_j$ augmentation is computationally expensive and yields no benefit [S2].

### Synthetic-NeRF View Space and Sampling
This variant defines the view space and sampling strategy for active viewpoint generation in synthetic NeRF scenes.

**Inputs:**
- Scene geometry or training set bounds.
- Target object of interest.

**Outputs:**
- A set of candidate viewpoints for active sampling.

**Steps:**
1. **Define View Space:**
   - **For Synthetic-NeRF scenes:** Define view space as a hemisphere with maximum radius $R_{max} = 4$. Use spherical coordinates for viewpoint representation. Constrain orientation to point towards the center of the object [S3, unofficial].
   - **For Tanks & Temple scenes:** Define view space as a 3D bounding box derived from the raw training set corners. Represent viewpoint state by 6DoF $se(3)$ [S3, unofficial].
2. **Sample Waypoints:**
   - Generate a Bézier trajectory within the view space.
   - Sample $N_{waypoints} = 100$ waypoints uniformly from this trajectory [S3, unofficial].
3. **Select Viewpoints for Information Gain:**
   - From the 100 waypoints, select $N_{views} = 10$ viewpoints for information gain calculation [S3, unofficial].
4. **Sample 3D Points for Calculation:**
   - Instead of sampling 3D points from all pixels, sample from $1/100$ of the pixels to reduce computation cost [S3, unofficial].

**Parameters:**
- **View Space Shape:** Hemisphere (Synthetic-NeRF) or 3D Bounding Box (Tanks & Temple) [S3, unofficial].
- **Max Radius:** 4 (for hemisphere) [S3, unofficial].
- **Waypoint Count:** 100 [S3, unofficial].
- **Info Gain Viewpoint Count:** 10 [S3, unofficial].
- **Pixel Sampling Ratio:** 1/100 [S3, unofficial].

**Pitfalls:**
- **Coordinate Representation:** Spherical coordinates only represent position, not rotation. Orientation must be explicitly constrained to point at the object center [S3, unofficial].

## Subtopics

### [Replica Dataset Preparation](12080-replica-dataset-preparation.md)
Replica Dataset Preparation transforms raw Replica synthetic indoor scan data into a clean, semantically annotated, and geometrically consistent dataset. It solves problems of unrealistic camera poses (e.g., inside geometry), label ambiguity, and inconsistent coordinate systems. It is distinct from general synthetic generation by focusing on post-processing and filtering of existing high-fidelity scans rather than procedural generation [12080-replica-dataset-preparation.md].

### Dataset Generation and Augmentation
A procedure to convert sensor poses and range measurements into a dataset $D$ suitable for training the SDDF model, including augmentation for sign balance. It specifically addresses the imbalance in sign indicators and SDDF signs by generating negative samples behind the observed surface. It avoids augmenting intersection indicators due to inefficiency and lack of benefit [S2].

### Synthetic-NeRF View Space and Sampling
Defines the camera view space and sampling strategy for active viewpoint generation and information gain calculation in synthetic NeRF scenes. It distinguishes between hemisphere-based view spaces for synthetic objects and bounding-box-based spaces for real-world scenes like Tanks & Temple, using specific sampling ratios to optimize computation [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of camera poses $N$ and the resolution of rendered images. Rendering is the dominant cost, typically parallelized across pixels or frames. SDDF augmentation is linear in the number of samples $O(M)$. Active sampling involves calculating information gain over $N_{views}$ viewpoints, which is costly but reduced by sampling only $1/100$ of pixels [S3, unofficial].
- **Memory Complexity:** Proportional to the number of samples in the dataset. SDDF dataset size doubles due to negative sample augmentation [S2].
- **Strengths:** Provides ground-truth data without sensor noise or calibration errors (before intentional noise injection). Allows for controlled variation in lighting, geometry, and camera paths.
- **Limitations:** Synthetic data may not perfectly match real-world sensor characteristics (domain gap). Requires manual selection of control points or scene parameters.
- **When to Use:** When real-world paired data is unavailable or too expensive to collect. When training models that require precise ground-truth geometry or depth.
- **When Not to Use:** When the target application is highly sensitive to real-world sensor artifacts that are difficult to simulate accurately. When computational resources for rendering are limited.

## Background
Synthetic dataset generation has become essential in neural rendering and 3D reconstruction due to the scarcity of large-scale, high-quality paired RGB-D datasets. Early works used synthetic scenes to validate algorithms before applying them to real data. The method described in [S1] (Neural RGB-D Surface Reconstruction) uses BlenderProc to generate synthetic datasets with Catmull-Rom splines for camera trajectories, applying noise to simulate real sensors. This approach allows for fair comparison between methods by providing consistent initial pose estimates (e.g., from BundleFusion) [S1]. The SDDF approach [S2] extends this by defining specific data structures and augmentation strategies for directional distance functions. The Synthetic-NeRF approach [S3, unofficial] further refines this for active mapping, defining view spaces and sampling strategies for information gain calculation.

## Sources
[S1] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
[S2] Learning Scene-Level Signed Directional Distance Function With Ellipsoidal Priors and Neural Residuals., https://doi.org/10.1109/tpami.2026.3688658, fetched 2026-10-26
[S3] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
