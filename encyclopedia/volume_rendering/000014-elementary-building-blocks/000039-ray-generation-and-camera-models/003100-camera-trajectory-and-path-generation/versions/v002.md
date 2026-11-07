# Camera Trajectory and Path Generation

Also called: Camera Path Generation, Trajectory Generation

## Overview
Camera Trajectory and Path Generation defines the sequence of camera poses (positions and orientations) used to sample rays for volume rendering or novel view synthesis. It solves the problem of determining where to place virtual cameras to capture a scene from desired viewpoints, ranging from static orbits to complex helical fly-throughs. It differs from [Ray generation and camera models](770-ray-generation-and-camera-models.md) by focusing on the temporal/spatial sequence of poses rather than the instantaneous ray construction, and from [Novel View Synthesis Rendering](6402-novel-view-synthesis-rendering.md) by excluding the actual rendering and encoding steps, providing only the geometric path.

## Shared principles
The shared principles of this family are the geometric definition of rays and the parallel execution model.

**Ray Definition**
Rays are defined by an origin $\mathbf{o}$ and a direction $\mathbf{d}$. The direction is typically normalized. The origin is often the camera eye position, but can vary based on the camera model.

**Parallel Execution**
Computations are mapped to parallel threads. Each thread typically processes one or more pixels or samples. Memory access patterns are optimized for coalescing. Synchronization is used to manage data dependencies between threads.

## Subtopics

### [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md)
Spiral Camera Path Generation creates a sequence of camera poses along a helical trajectory around a scene's centroid for novel view synthesis. It solves the problem of generating smooth, continuous fly-through or orbiting viewpoints that cover 360° horizontal rotation while introducing vertical variation, which prevents the "flat" appearance of pure circular orbits. It differs from [Circular Camera Path Generation](2734-circular-camera-path-generation.md) by adding a vertical oscillation component (z-rate), making it a non-planar helix rather than a degenerate planar circle. It differs from [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md) by assuming a central focal point and isotropic or axis-aligned radii rather than following a specific linear or curved structural axis.

### [Elongated Structure Camera Trajectory Generation](4666-elongated-structure-camera-trajectory-generation.md)
This method generates multi-view camera poses for elongated structures (e.g., walls, ancient city fortifications) from a 3D point cloud by fitting a principal axis via Principal Component Analysis (PCA), sampling stations along it, and validating positions against a density grid. It replaces indoor room-centric hardcoded poses with a data-driven approach suitable for large-scale outdoor scenes. Unlike [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md) which assumes a central object, this method aligns the camera path with the dominant linear feature of the scene.

### [Novel View Synthesis Rendering](6402-novel-view-synthesis-rendering.md)
Novel View Synthesis Rendering generates a continuous video sequence of a 3D scene from a trained implicit representation (typically a Neural Radiance Field, or NeRF) by traversing a predefined camera trajectory. It solves the problem of visualizing a static 3D reconstruction as a dynamic 360° experience. Unlike static rendering methods that produce a single image, this method outputs a time-ordered sequence of frames. It is distinct from trajectory generation methods (which only compute poses) by including the actual ray casting, volume rendering, and video encoding steps.

### [Sparse Trajectory Generation](7215-sparse-trajectory-generation.md)
Generates a sparse trajectory by selecting specific frames from a full trajectory based on a mapping file.

### [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md)
Generates a sequence of rendered images by rotating the camera around a specified axis and capturing frames for video export.

### [Synthetic Camera Trajectory Generation](7942-synthetic-camera-trajectory-generation.md)
Not given in the sources.

### Elliptical Camera Path Generation
**Definition**
Generates an elliptical render path based on given camera poses, interpolating positions and orientations [S1, unofficial].

**Steps**
1. Calculate the focal point center using `focus_point_fn(poses)` [S1, unofficial].
2. Set the path height offset to $z=0$: $\text{offset} = [\text{center}[0], \text{center}[1], 0]$ [S1, unofficial].
3. Compute scaling factors $\text{sc}$ using the 90th percentile of absolute distances from offset [S1, unofficial].
4. Define ellipse bounds $\text{low} = -\text{sc} + \text{offset}$ and $\text{high} = \text{sc} + \text{offset}$ [S1, unofficial].
5. Compute $z$ bounds using 10th and 90th percentiles of pose $z$-coordinates [S1, unofficial].
6. Define `get_positions(theta)` to interpolate $x$-$y$ using $\cos/\sin$ and optionally $z$ using $\cos$ with phase shift [S1, unofficial].
7. Generate $\theta$ from $0$ to $2\pi$ for $n\_frames+1$ points, compute positions, and discard the last duplicate point [S1, unofficial].
8. Determine the path's up vector by averaging input pose up vectors and selecting the axis with the largest absolute value [S1, unofficial].
9. Construct and return view matrices for each position using `viewmatrix(p - center, up, p)` [S1, unofficial].

### Photometric Initialization
**Definition**
An initialization method for the transformation $T \in SE(3)$ between Scene A and Scene B. It generates good camera viewpoints for Scene A using its Viewshed Field (VFA), transforms them by candidate $T$, and scores them based on their viewshed likelihood in Scene B (VFB) [S2].

**Steps**
1. Generate a set of $N$ good camera viewpoints $C_A$ for Scene A using VFA [S2].
2. For a candidate transformation $T$, transform each camera $C_i$ in $C_A$ to get $T(C_i)$. Calculate the viewshed score for all oriented points $p$ originating from these transformed cameras in Scene B [S2].
3. Sample multiple random transformations $T$. Select the $T$ with the highest $Score_{init}$ as the initialization [S2].

**Formula**
Calculate the initialization score for $T$ as the median over all transformed cameras of the sum of viewshed values in Scene B:
$$ Score_{init}(C_A; T) = \text{Median}_{C_A}\left(\sum_{p \in T(C_i)} VF_B(p)\right) $$
[S2]

### Multi-timestep Sampling for Scene Generation
**Definition**
A method proposing multi-timestep sampling for multi-stage scene-level generation [S3].

**Steps**
1. Multi-timestep sampling is used for multi-stage scene-level generation [S3].
2. Surrounding environment, ground, and objects are synthesized [S3].

### Synthetic Dataset Generation
**Steps**
1. Define camera trajectory using Catmull-Rom spline interpolation on manually selected control points [S4].
2. Render color and depth images for each camera pose along the trajectory using BlenderProc [S4].
3. Apply noise to depth maps to simulate real sensor noise. Use provided noisy depth/color for ICL scene instead of rendering [S4].

**Parameters**
Initial pose estimation: BundleFusion [14] provides initial camera trajectory estimates for all methods except COLMAP to ensure fair comparison [S4].

## Background
The generation of camera trajectories is fundamental to multi-view stereo, Structure from Motion (SfM), and Neural Radiance Fields (NeRF). Early methods relied on fixed geometric paths (orbits, spirals) derived from the statistical properties of training data, such as the centroid and extent of camera positions [S1, unofficial][S3, S4]. For elongated structures, data-driven approaches using Principal Component Analysis (PCA) on point clouds were introduced to align trajectories with scene geometry [S1, unofficial]. In the context of NeRF and 3D Gaussian Splatting, spiral paths became standard for novel view synthesis to ensure smooth 360° coverage with vertical variation [S1, unofficial][S3, S4]. Photometric initialization methods using Viewshed Fields (VF) were developed to robustly initialize camera poses for rigid registration between scenes [S2]. Synthetic dataset generation often employs Catmull-Rom splines for smooth trajectory definition [S4].

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] VF-NeRF: Viewshed Fields for Rigid NeRF Registration, https://doi.org/10.1007/978-3-031-73027-6_10, fetched 2026-10-26
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S4] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
