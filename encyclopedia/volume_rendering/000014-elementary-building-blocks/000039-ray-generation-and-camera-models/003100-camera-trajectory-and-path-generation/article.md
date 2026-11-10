# Camera Trajectory and Path Generation

Also called: Camera Path Generation, Trajectory Sampling, Novel View Trajectory

## Overview
This family defines methods for generating sequences of camera poses (position and orientation) for volume rendering, novel view synthesis, and 3D reconstruction. It solves the problem of determining valid viewing geometries when real-world capture data is unavailable, insufficient, or requires optimization for specific computational constraints. What sets this family apart from sibling methods like [Sky Gradient](1262-sky-gradient.md) or [Arcball Camera](1639-arcball-camera.md) is the focus on generating *temporal sequences* or *structured multi-view sets* rather than interactive single-view controls or static background models. It differs from [Camera Ray Generation](7456-camera-ray-generation.md) by operating at the pose level (defining the camera's world transform) rather than the ray level (defining individual pixel rays), although it feeds directly into ray generation.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model [G].

**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$ [G]. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model [G].

**Parallel Execution**
Computations are mapped to parallel threads [G]. Each thread typically processes one or more pixels or samples [G]. Memory access patterns are optimized for coalescing [G]. Synchronization is used to manage data dependencies between threads [G].

## Subtopics

### [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md)
Generates multi-view camera poses for large-scale outdoor elongated structures (e.g., ancient city walls) from a 3D point cloud. It fits a principal axis via Principal Component Analysis (PCA) to the point cloud's XY coordinates, samples stations along this axis, and validates positions against a density grid to ensure coverage. It replaces indoor room-centric hardcoded poses with data-driven trajectories aligned to the structure's geometry. It differs from [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) by following a linear principal axis rather than a helical path, and from [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md) by maintaining a fixed distance from a linear centerline rather than orbiting a central point.

### [Novel View Synthesis Rendering](6402-novel-view-synthesis-rendering.md)
Generates a continuous video of novel views by traversing a predefined camera trajectory (typically a spiral path) and rendering each frame using a trained Neural Radiance Field (NeRF). It differs from static novel view synthesis by outputting a temporal sequence (MP4) rather than a single image, and from diffusion-based methods like iNVS by relying on volumetric ray marching through a learned implicit representation rather than warping and inpainting.

### [Sparse Trajectory Generation](7215-sparse-trajectory-generation.md)
Reduces the computational load of trajectory-based rendering methods by selecting a subset of frames from a full trajectory based on a predefined mapping file. It serves as a preprocessing step for algorithms like 2D Gaussian Splatting, allowing for efficient training or rendering on hardware with limited memory or compute resources by skipping redundant or intermediate camera poses. Unlike continuous path generation methods such as [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) or [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md), this method operates on discrete, existing trajectory data, acting as a filter rather than a generator.

### [Synthetic Camera Trajectory Generation](7942-synthetic-camera-trajectory-generation.md)
Creates a sequence of camera poses for volume rendering or novel view synthesis when real-world capture data is unavailable or insufficient. It solves the problem of defining valid viewing geometries by sampling positions and orientations from a structured distribution, typically using low-discrepancy sequences like Sobol samplers to ensure uniform coverage of the scene volume. It differs from spiral or orbit generation by decoupling position and direction sampling to allow arbitrary spherical coverage, and from random pose selection by providing deterministic, space-filling trajectories rather than stochastic samples.

### [Synthetic Dataset Generation](9337-synthetic-dataset-generation.md)
Creates training data for volume rendering and 3D reconstruction models by simulating sensor observations in computer-generated environments. It solves the problem of lacking real-world labeled data by providing ground-truth geometry, semantic labels, and controlled noise characteristics. Unlike methods that generate camera trajectories for active planning or reconstruct scenes from sparse inputs, this method focuses on the pipeline of rendering, augmenting, and cleaning synthetic data to match the statistical properties of real sensors. It differs from [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md) by prioritizing data quality and label correctness over trajectory coverage, and from [Novel View Synthesis Rendering](6402-novel-view-synthesis-rendering.md) by providing explicit geometric and semantic ground truth rather than just photometric consistency.

### [Parametric Path Generation](11476-parametric-path-generation.md)
Constructs camera trajectories by evaluating explicit mathematical functions of a scalar parameter (typically angle or time) to produce a sequence of camera poses. It solves the problem of generating smooth, continuous novel views from a sparse set of training images by defining the geometry of the path analytically rather than interpolating between discrete keyframes. It differs from [Sparse Trajectory Generation](7215-sparse-trajectory-generation.md) by relying on closed-form equations for position and orientation, and from [Synthetic Camera Trajectory Generation](7942-synthetic-camera-trajectory-generation.md) by deriving parameters directly from scene statistics (e.g., camera bounding boxes) rather than arbitrary artistic constraints.

### Photometric Initialization
An initialization method for the transformation $T \in SE(3)$ between Scene A and Scene B. It generates good camera viewpoints for Scene A using its Viewshed Field (VFA), transforms them by candidate T, and scores them based on their viewshed likelihood in Scene B (VFB) [S1].
- Generate a set of N good camera viewpoints $C_A$ for Scene A using VFA [S1].
- For a candidate transformation T, transform each camera $C_i$ in $C_A$ to get $T(C_i)$. Calculate the viewshed score for all oriented points p originating from these transformed cameras in Scene B [S1].
- Sample multiple random transformations T. Select the T with the highest $Score_{init}$ as the initialization [S1].
- Calculate the initialization score for T as the median over all transformed cameras of the sum of viewshed values in Scene B: $Score_{init}(C_A; T) = Median_{C_A}(\sum_{p \in T(C_i)} VF_B(p))$ [S1].

### Multi-timestep Sampling for Scene Generation
A method proposing multi-timestep sampling for multi-stage scene-level generation [S2].
- Multi-timestep sampling is used for multi-stage scene-level generation [S2].
- Surrounding environment, ground, and objects are synthesized [S2].

### Stereoscopic Camera Path Generation
A method to generate two distinct camera paths for stereoscopic visualization by calculating a second eye position based on a separation angle and the primary view vector [S3].
- Define the primary camera position c, look-at point l, and sky vector s [S3].
- Calculate the viewing vector v = l - c [S3].
- Calculate the right direction vector r using the cross product: r = S × v [S3].
- Calculate the orthogonal component S of the sky vector s using: $S = s - \cos(\alpha) v \frac{|s|}{|v|}$, where $\alpha$ is the angle between s and v [S3].
- Calculate $\cos(\alpha)$ using: $\cos(\alpha) = \frac{v \cdot s}{|v| |s|}$ [S3].
- Calculate the right eye position C using: $C = c + \frac{r}{|r|} \frac{\beta 2\pi}{360} |v|$, where $\beta$ is the separation angle [S3].
- The sky vector s does not need to be orthogonal to the view vector v for Splotch, but orthogonalization is performed for the stereo calculation [S3].

### B-Spline Camera Path Interpolation
Creates a smooth spline path between input keyframe camera poses using multidimensional B-spline interpolation [S4, unofficial].
- Convert poses to (position, lookat, up) format using poses_to_points(poses, dist=rot_weight) [S4, unofficial].
- Run multidimensional B-spline interpolation via scipy.interpolate.splprep on reshaped points with degree k and smoothness s [S4, unofficial].
- Evaluate spline at n_interp*(points.shape[0]-1) points using splev [S4, unofficial].
- Reshape back to (n, 3, 3) format [S4, unofficial].
- Convert back to pose matrices using points_to_poses(new_points) [S4, unofficial].
- Parameters: n_interp (interpolation factor per segment), spline_degree (polynomial degree, default 5), smoothness (smoothing factor, default 0.03), rot_weight (relative weighting of rotation/translation, default 0.1) [S4, unofficial].

### Temporal Camera Selection with Smooth Fading
- Initialize camera selection scores: selected cameras get s=1.0, others get s=0.0 [S5].
- Apply temporal filtering to scores $s_i^t$ to get weights $w_i^t$ using the exponential moving average formula with $\lambda=0.05$ [S5].
- Select the N views with the highest updated weights $w_i^t$ for rendering [S5].
- Refine weights across all views using element-wise subtraction of the minimum weight, followed by normalization, to ensure zero weight for views entering/leaving the set and move C0 discontinuity to C1 [S5].
- Formula: $w_i^t = \lambda s_i^t + (1.0 - \lambda) w_i^{t-1}$ [S5].
- Formula: $w^{CS} = \frac{w - \min_i w_i}{\sum (w - \min_i w_i)}$ [S5].
- Parameters: $\lambda$ (temporal filtering adaptation speed, typical 0.05), N (number of highest weighted views, typical 9) [S5].
- Pitfall: Temporal instability when cameras appear/disappear. Handled by smooth fading and weight refinement to ensure gradients of weights become discontinuous instead of actual values [S5].

## Background
Camera trajectory generation is fundamental to novel view synthesis (NVS) and 3D reconstruction, where the goal is to render views from angles not captured during data acquisition. Early methods relied on manual keyframe definition or simple geometric paths (spirals, orbits) [S1, S3][S4, unofficial]. With the advent of Neural Radiance Fields (NeRF) and 3D Gaussian Splatting, the demand for high-quality, dense, and uniformly distributed camera poses increased to ensure robust training and rendering [S1, S2]. Methods like Synthetic Camera Trajectory Generation use low-discrepancy sequences (Sobol) to avoid clustering artifacts common in random sampling [S1]. For specific applications like stereoscopic visualization, specialized paths are required to maintain correct interocular geometry [S3]. In large-scale outdoor scenarios, data-driven approaches using PCA on point clouds have been developed to align trajectories with elongated structures [S1]. Furthermore, computational efficiency has led to sparse trajectory selection methods to reduce training time for large datasets [S1].

## Sources
[S1] VF-NeRF: Viewshed Fields for Rigid NeRF Registration, https://doi.org/10.1007/978-3-031-73027-6_10, fetched 2026-10-26
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S3] Splotch: visualizing cosmological simulations, https://doi.org/10.1088/1367-2630/10/12/125006, fetched 2026-10-28
[S4] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
[S5] Point-Based Neural Rendering with Per-View Optimization, https://www.youtube.com/watch?v=TupOewenSAk, fetched 2026-10-31
