# Wolf et al. Stereo Method
Also called: Stereo 3DGS, Stereo-calibrated 3DGS

## Overview
The Wolf et al. Stereo Method is a post-processing pipeline for 3D Gaussian Splatting (3DGS) that converts the discrete, volumetric representation into a continuous, watertight triangular mesh. It solves the problem of poor geometric reconstruction quality inherent in standard 3DGS by leveraging novel view synthesis to generate stereo-calibrated image pairs, extracting dense depth maps via stereo matching, and fusing them into a mesh using Truncated Signed Distance Function (TSDF) integration [S1, S2]. Unlike methods that modify the Gaussian representation itself (e.g., [2D-GS](10145-representation-and-kernel-modifications.md)) or rely on implicit neural fields, this approach remains agnostic to the underlying 3DGS structure, treating it merely as a high-fidelity renderer for stereo pairs [S1].

## How it works
The method operates in three sequential stages: stereo view generation, stereo depth estimation, and TSDF fusion.

### Inputs and Outputs
- **Inputs**: A trained 3DGS model (Gaussians with position, covariance, opacity, and spherical harmonic coefficients), a set of training images covering the region of interest, and the scene radius/diameter.
- **Outputs**: A triangular mesh representing the scene surface.

### Data Structures
The 3DGS model is assumed to be fully optimized and stored in a format compatible with the standard 3DGS rasterizer. No additional data structures are introduced by the method itself; it relies on external stereo matching algorithms and TSDF volume implementations.

### Steps

1. **Stereo View Generation**
   - Select a training image pose $(R_L, T_L)$ as the left camera.
   - Compute the right camera pose $(R_R, T_R)$ using a horizontal baseline $b$.
   - Render the left and right images using the trained 3DGS model.

2. **Stereo Depth Estimation**
   - Input the rendered left and right image pairs into a stereo matching algorithm.
   - The method cites DLNR [S2] as the preferred algorithm, but any state-of-the-art neural stereo matching model is applicable.
   - Output dense depth maps for each view.

3. **TSDF Fusion**
   - Aggregate all extracted depth maps into a 3D volume.
   - Perform Truncated Signed Distance Function (TSDF) integration to fuse the depths.
   - Extract the zero-level set from the TSDF volume to generate the triangular mesh.

### Formulas

**Stereo Camera Pose**
The right camera pose is derived from the left camera pose $(R_L, T_L)$ and a horizontal baseline $b$:

$$
R_R = R_L
$$

$$
T_R = T_L + (R_L \times [b, 0, 0])
$$

Where:
- $R_L, R_R \in \mathbb{R}^{3 \times 3}$ are the rotation matrices for the left and right cameras.
- $T_L, T_R \in \mathbb{R}^3$ are the translation vectors for the left and right cameras.
- $b$ is the horizontal baseline distance.
- $[b, 0, 0]$ is the baseline vector in the local coordinate system of the left camera, rotated by $R_L$ to the world frame.

**Baseline Parameter**
The baseline $b$ is set to 7% of the scene radius (or 3.5% of the scene diameter) [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wolf_stereo_method(trained_3dgs, training_images, scene_radius):
    """
    Generates a triangular mesh from a trained 3DGS model using stereo depth fusion.
    
    Args:
        trained_3dgs: Optimized 3D Gaussian Splatting model.
        training_images: List of training image poses and data.
        scene_radius: Radius of the scene bounding volume.
        
    Returns:
        mesh: Triangular mesh object.
    """
    # Step 1: Generate Stereo Views
    # Select a training image pose as the left camera
    left_pose = training_images[0].pose  # (R_L, T_L)
    
    # Calculate baseline
    baseline = 0.07 * scene_radius
    
    # Calculate right camera pose
    # R_R = R_L
    # T_R = T_L + (R_L * [baseline, 0, 0])
    right_pose = compute_stereo_pose(left_pose, baseline)
    
    # Render left and right images
    left_image = trained_3dgs.render(left_pose)
    right_image = trained_3dgs.render(right_pose)
    
    # Step 2: Stereo Depth Estimation
    # Use a stereo matching algorithm (e.g., DLNR)
    # Note: The specific implementation of the stereo matcher is external
    depth_map_left, depth_map_right = stereo_match(left_image, right_image)
    
    # Step 3: TSDF Fusion
    # Initialize TSDF volume
    tsdf_volume = initialize_tsdf_volume(scene_radius)
    
    # Fuse depth maps into TSDF volume
    # This step typically involves projecting depth maps into 3D and updating the volume
    fuse_depth_into_tsdf(tsdf_volume, depth_map_left, left_pose)
    fuse_depth_into_tsdf(tsdf_volume, depth_map_right, right_pose)
    
    # Extract mesh from TSDF volume
    mesh = extract_mesh_from_tsdf(tsdf_volume)
    
    return mesh

def compute_stereo_pose(left_pose, baseline):
    """
    Computes the right camera pose given the left pose and baseline.
    """
    R_L, T_L = left_pose
    # Rotate the baseline vector [b, 0, 0] by R_L
    baseline_vector = R_L @ np.array([baseline, 0, 0])
    T_R = T_L + baseline_vector
    R_R = R_L
    return (R_R, T_R)
```

### Parameters and Edge Cases
- **Baseline ($b$)**: Set to 7% of the scene radius. This provides a "sweet spot" in the range of 14% to 70% of the scene diameter, balancing disparity visibility and occlusion [S2].
- **Coverage**: A sufficient amount of images covering the region of interest must be used to ensure sufficient coverage and avoid noisy artifacts [S2].
- **Pose Selection**: The left image of the stereo pair should be at the same pose as a training image to stay close to training data, resulting in cleaner renders [S2].
- **Stereo Matching Algorithm**: DLNR with pre-trained Middlebury weights is cited as providing the best qualitative and quantitative results [S2].

## Subtopics

### Stereo View Generation for 3DGS
This variant focuses on the specific strategy for generating stereo-calibrated views from a single-camera 3DGS reconstruction. It sets the left camera pose to match a training image pose $(R_L, T_L)$ and computes the right pose with a horizontal baseline $b$ as $R_R = R_L$ and $T_R = T_L + (R_L \times [b, 0, 0])$. The baseline $b$ is set to 7% of the scene radius to optimize the disparity range. This approach leverages the novel view synthesis capabilities of 3DGS to effectively convert a monocular capture into a stereo capture [S2].

## Operating characteristics
- **Time Complexity**: Dominated by the rendering of stereo views and the stereo matching process. The TSDF fusion is generally linear in the number of depth pixels and volume voxels.
- **Memory Complexity**: Requires storing the TSDF volume, which scales with the volume of the scene and the chosen resolution.
- **Strengths**: Produces watertight, smooth triangular meshes; leverages state-of-the-art stereo matching algorithms; agnostic to the 3DGS representation details.
- **Limitations**: Relies on the quality of the 3DGS renderings; noisy or incomplete 3DGS reconstructions can lead to poor depth estimates; the discrete nature of 3DGS may limit geometric accuracy compared to implicit methods [S1].
- **When to use**: When a high-quality triangular mesh is required from a 3DGS reconstruction, and the scene is well-covered by training images.
- **When not to use**: When the scene has large unobserved regions or when the 3DGS reconstruction is noisy due to insufficient training data.

## Background
The method was introduced by Wolf et al. in the paper "GS2Mesh: Surface Reconstruction from Gaussian Splatting via Novel Stereo Views" [S2]. It addresses the limitation of 3DGS in producing explicit geometry, which is often noisy or incomplete due to the discrete nature of the Gaussian representation [S1]. By using 3DGS as a renderer for stereo pairs, the method bypasses the need for explicit geometry optimization during training, instead relying on post-processing for surface extraction. This approach has been cited as a significant advancement in geometry reconstruction for 3DGS [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] GS2Mesh: Surface Reconstruction from Gaussian Splatting via Novel Stereo Views, https://doi.org/10.1007/978-3-031-73024-5_13, fetched 2026-10-29
