# Structure-from-Motion (SfM)

Also called: Agisoft Metashape pipeline, MVS reconstruction, Photogrammetric Procedure

## Overview
Structure-from-Motion (SfM) is a traditional image-based reconstruction pipeline that estimates camera parameters (extrinsics and intrinsics) to build sparse and dense point clouds, followed by mesh generation and texture extraction [S1, S2]. It solves the problem of recovering 3D geometry and camera poses from a set of uncalibrated photographs, providing the essential spatial prior for subsequent neural rendering methods like NeRF and Gaussian Splatting [S3][S4, unofficial]. Unlike its siblings, [Canonical Frame Estimation via Facial Landmarks](16399-canonical-frame-estimation-via-facial-landmarks.md) which anchors to semantic features, and [Deep Image Matching Initialization](16400-deep-image-matching-initialization.md) which uses learned backbones, SfM relies on traditional geometry-based multi-view stereo techniques, such as SIFT feature matching and RANSAC verification, to produce explicit geometry [S1][S4, unofficial].

## How it works
**Inputs:** A set of uncalibrated photographs or video frames [S4, unofficial].
**Outputs:** Camera poses (extrinsics $[R|t]$), camera intrinsics (focal length, principal point, distortion), and a sparse 3D point cloud [S4, unofficial].

**Data Structures:**
*   **Keypoint:** A 2D coordinate $(u, v)$ and a descriptor vector $d \in \mathbb{R}^k$ (typically 128 dimensions for SIFT) summarizing local pixel intensity gradients [S4, unofficial].
*   **Match:** A pair of keypoints $(k_i, k_j)$ from images $I_i$ and $I_j$ with descriptor distance below a threshold.
*   **Point Cloud:** A set of 3D points $P = \{p_1, \dots, p_N\}$ where each $p_n \in \mathbb{R}^3$ and has a color attribute [S1].
*   **Camera Pose:** A $3 \times 4$ matrix $[R|t]$ representing rotation and translation [S2].

**Steps:**

1.  **Feature Extraction:** Detect distinctive patches (corners, spots, texture) in each image using a detector like SIFT. SIFT is monochrome, discarding hue and saturation [S4, unofficial]. Compute a descriptor vector for each keypoint that summarizes surrounding pixels, surviving rotation, scale, and brightness changes [S4, unofficial].
2.  **Pairwise Matching:** Compare descriptors between images. For large sets, use a vocabulary tree to avoid comparing all $N^2$ pairs [S4, unofficial]. Identify candidate matches where descriptor distance is minimal [S4, unofficial].
3.  **Geometric Verification:** Apply RANSAC (Random Sample Consensus) to discard incorrect matches [S4, unofficial]. RANSAC guesses a camera relationship (fundamental matrix if uncalibrated, essential matrix if calibrated) from a random subset of matches, checks agreement with remaining matches, and keeps the guess with the most inliers [S4, unofficial]. True matches must lie on a specific epipolar line in the other image [S4, unofficial].
4.  **Incremental Reconstruction:**
    *   Select a good starting pair with a wide baseline and many verified matches [S4, unofficial].
    *   Reconstruct the two cameras and triangulate their shared 3D points. Triangulation finds the closest approach of rays from camera centers through feature locations [S4, unofficial].
    *   Register new cameras using Perspective-n-Point (PnP) based on known 3D points and their 2D projections in the new image [S4, unofficial].
    *   Triangulate new points added by the registered camera [S4, unofficial].
    *   Repeat until all images are registered [S4, unofficial].
5.  **Bundle Adjustment (BA):** Run periodically to minimize total reprojection error [S4, unofficial]. Optimize all camera poses and 3D points simultaneously [S4, unofficial]. The objective function is:
    $$ E = \sum_{i} \sum_{j} \| \mathbf{u}_{ij} - \pi(P_j, \mathbf{C}_i) \|^2 $$
    where $\mathbf{u}_{ij}$ is the observed 2D keypoint, $P_j$ is the 3D point, $\mathbf{C}_i$ is the camera pose, and $\pi$ is the projection function [S4, unofficial].
6.  **Self-Calibration:** Treat intrinsics (focal length, principal point, lens distortion) as unknowns to be recovered via BA, rather than relying solely on EXIF metadata [S4, unofficial]. If photos come from the same physical camera with a fixed lens, intrinsics are shared across the whole set to better constrain the solution [S4, unofficial].
7.  **Dense Reconstruction (Optional):** Generate a dense point cloud, create a 3D mesh, and extract textures [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
MIN_MATCHES = 50
BA_INTERVAL = 10

def structure_from_motion(images):
    # Step 1: Feature Extraction
    keypoints = {}
    for img_id, img in images.items():
        keypoints[img_id] = sift_detect_and_describe(img)
        
    # Step 2: Pairwise Matching
    matches = {}
    for i, img_i in enumerate(images):
        for j, img_j in enumerate(images):
            if i >= j: continue
            candidates = match_descriptors(keypoints[img_i], keypoints[img_j])
            # Step 3: Geometric Verification via RANSAC
            inliers = ransac_epipolar_geometry(candidates)
            if len(inliers) > MIN_MATCHES:
                matches[(img_i, img_j)] = inliers
                
    # Step 4: Incremental Reconstruction
    init_pair = select_best_pair(matches)
    cam1, cam2 = reconstruct_pair(init_pair, matches[init_pair])
    points = triangulate(cam1, cam2, matches[init_pair])
    registered = {init_pair[0], init_pair[1]}
    cameras = {init_pair[0]: cam1, init_pair[1]: cam2}
    
    while len(registered) < len(images):
        next_img = find_best_candidate(images, registered, matches)
        pose = solve_pnp(points, next_img, matches[(next_img, r)])
        cameras[next_img] = pose
        new_points = triangulate_new_points(pose, points)
        points.extend(new_points)
        registered.add(next_img)
        
        if len(registered) % BA_INTERVAL == 0:
            optimize_bundle_adjustment(cameras, points)
            
    return cameras, points

def sift_detect_and_describe(img):
    return [(u, v, d) for u, v, d in img.detect_sift()]

def match_descriptors(kps1, kps2):
    return [(i, j, np.linalg.norm(kps1[i][2] - kps2[j][2])) for i in range(len(kps1)) for j in range(len(kps2))]

def ransac_epipolar_geometry(candidates):
    best_inliers = []
    for _ in range(100):
        subset = random.sample(candidates, 7)
        model = fit_epipolar_model(subset)
        inliers = [c for c in candidates if reprojection_error(c, model) < THRESHOLD]
        if len(inliers) > len(best_inliers):
            best_inliers = inliers
    return best_inliers

def select_best_pair(matches):
    return max(matches.keys(), key=lambda k: len(matches[k]))

def reconstruct_pair(pair, inliers):
    return estimate_relative_pose(inliers)

def triangulate(cam1, cam2, inliers):
    return [triangulate_point(cam1, cam2, m) for m in inliers]

def find_best_candidate(images, registered, matches):
    return max(set(images.keys()) - registered, key=lambda img: sum(len(matches.get((img, r), [])) + len(matches.get((r, img), [])) for r in registered))

def solve_pnp(points, next_img, matches):
    return solve_pnp_solver(points, next_img, matches)

def triangulate_new_points(pose, points):
    return [triangulate_point(pose, p) for p in points if visible_in_view(p, pose)]

def optimize_bundle_adjustment(cameras, points):
    minimize_reprojection_error(cameras, points)
```

**Parameters:**
*   **Quality Mode:** Set to 'high quality' for alignment, dense point cloud, mesh, and texture generation to maximize fidelity [S1].
*   **Motion Trajectory Segmentation Interval:** In progressive motion sequences, segment every five frames to introduce a local prior ensuring the start pose of the next segment is close to the end pose of the previous segment: $[R|t]_{p+1} \leftarrow [R|t]_p$ [S2].
*   **Baseline:** Sufficient separation between camera positions is required for stable triangulation [S1].

**Edge Cases and Pitfalls:**
*   **Zooming:** Creates two groups of images with different intrinsics, violating the shared-intrinsics assumption [S4, unofficial].
*   **Rolling Shutter:** Smears geometry during fast motion because top/bottom of frames are exposed at different times, violating the pinhole model assumption [S4, unofficial].
*   **Cropped/Resized Images:** Invalidate the principal point and focal length relative to the original sensor [S4, unofficial].
*   **Motion Blur:** Smears feature points, causing bad matches and poor triangulation. Mitigation: detect/drop blurriest frames and shoot at high shutter speeds [S4, unofficial].
*   **Tiny Baselines:** Processing too many nearly-identical video frames wastes compute and provides little new triangulation information [S4, unofficial].

## Subtopics

### [Canonical Frame Estimation via Facial Landmarks](16399-canonical-frame-estimation-via-facial-landmarks.md)
This method estimates a canonical coordinate frame for selfie and full-body captures using facial landmarks, enabling the automatic generation of novel view trajectories for NeRF training. It serves as a preprocessing step that aligns all input camera poses to a canonical coordinate system using 3D facial keypoints and similarity transformation. Unlike general Structure-from-Motion (SfM) alignment which relies on scene geometry, this method anchors the coordinate system to specific semantic features of the subject (the face), ensuring consistent orientation regardless of the initial camera pose.

### [Deep Image Matching Initialization](16400-deep-image-matching-initialization.md)
Deep Image Matching Initialization is a preprocessing pipeline that constructs a robust initial 3D geometry scaffold from uncalibrated input images, replacing traditional sparse Structure-from-Motion (SfM) methods. It solves the limitations of SfM and Multi-View Stereo (MVS) regarding robustness, efficiency, and scalability in challenging scenes. By leveraging deep learning backbones for pose and depth prediction, or deep feature matching for correspondence, it generates a dense, geometry-aware point cloud that serves as the foundation for subsequent neural rendering optimization.

### [Preprocessing and Initialization](18949-preprocessing-and-initialization.md)
Preprocessing and Initialization establishes the geometric foundation for volume rendering by estimating camera poses and generating an initial sparse or dense point cloud from input images. It solves the problem of providing accurate extrinsic parameters and a valid spatial prior for subsequent neural rendering or 3D reconstruction pipelines. What sets it apart from its siblings, [Canonical Frame Estimation via Facial Landmarks](16399-canonical-frame-estimation-via-facial-landmarks.md) and [Deep Image Matching Initialization](16400-deep-image-matching-initialization.md), is its reliance on traditional geometry-based multi-view stereo (MVS) techniques, such as feature matching and triangulation, often augmented with specific constraints like low-frequency masking or omnidirectional rectification, rather than purely learned canonical frames or deep matching networks.

### On-the-Fly SfM
Traditional SfM frameworks (e.g., COLMAP) are incapable of processing images in an online manner [S5]. On-the-Fly SfM employs a Hierarchical Weighted Local Bundle Adjustment (BA), in which only neighboring connected and registered images are incorporated into BA after acquiring a new image, significantly reducing time complexity compared to global BA [S5]. It achieves online feature matching via global feature retrieval on HNSW (Hierarchical Navigable Small World) to enable efficient dynamic updates for the matching matrix [S5].

### Camera Parameter Learning
A training strategy that enhances model robustness to camera parameters when small camera motion hinders standard SfM estimation (e.g., COLMAP) [S6]. It uses RCVD to predict initial camera parameters for the input video and simultaneously updates both camera parameters and MLPs during training to mitigate the impact of potentially inaccurate predictions [S6].

## Operating characteristics
*   **Time Complexity:** Incremental SfM scales poorly with the number of images due to $O(N^2)$ matching and expensive global BA [S4, unofficial]. COLMAP is 8x faster than HLOC for 310 frames and 26.5x faster for 1392 frames [S3]. LocalRF is significantly slower than both [S3].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On a machine with an RTX A6000 GPU and 128 GB RAM, COLMAP took 00:08:26 for 310 frames, 00:49:29 for 1392 frames, and 04:32:53 for 2784 frames [S3]. HLOC took 01:08:39 for 310 frames and 21:51:34 for 1392 frames [S3].
*   **Strengths:** Produces explicit geometry (point clouds and meshes) [S1]. Standard tool (COLMAP) is widely supported [S4, unofficial].
*   **Limitations:** Fails with sparse views (HLOC is better for sparse views) [S3]. Struggles with repetitive textures, reflections, and motion blur [S4, unofficial]. Not suitable for real-time online reconstruction without variants like On-the-Fly SfM [S5].
*   **When to Use:** When high-quality explicit geometry is needed, or as a seed for Gaussian Splatting/NeRF training with dense views [S4, unofficial].
*   **When Not to Use:** For real-time online applications (use On-the-Fly SfM [S5]), or when dealing with very sparse views where learned methods (HLOC) may perform better [S3].

## Background
Structure-from-Motion is a technique to recover camera poses, camera calibration, and a sparse 3D point cloud from a set of uncalibrated photographs [S4, unofficial]. It is described as 'load-bearing' for Gaussian Splatting because the splats require known camera positions to optimize against the source images [S4, unofficial]. The de facto standard tool is COLMAP, an open-source SfM and Multi-View Stereo package by Schönberger and Frahm (CVPR 2016) [S4, unofficial]. It is used as a standard pipeline for heritage documentation and comparative baseline evaluation against NeRFs [S1].

## Sources
[S1] Comparative Assessment of Neural Radiance Fields and Photogrammetry in Digital Heritage: Impact of Varying Image Conditions on 3D Reconstruction, https://doi.org/10.3390/rs16020301, fetched 2026-10-26
[S2] Neural Radiance Fields-Based 3D Reconstruction of Power Transmission Lines Using Progressive Motion Sequence Images., https://doi.org/10.3390/s23239537, fetched 2026-10-26
[S3] Towards Abdominal 3-D Scene Rendering from Laparoscopy Surgical Videos Using NeRFs, https://doi.org/10.1007/978-3-031-45673-2_9, fetched 2026-10-26
[S4] Gaussian Splats Part 2: From Photos to a Trained Model, https://alphapixeldev.com/gaussian-splats-part-2-creating-gaussian-splats/, fetched 2026-10-29 (unofficial)
[S5] Gaussian On-the-Fly Splatting: A Progressive Framework for Robust Near Real-Time 3DGS Optimization, https://doi.org/10.1109/lra.2025.3632729, fetched 2026-10-30
[S6] Dynamic View Synthesis From Small Camera Motion Videos, https://doi.org/10.1109/tvcg.2025.3565642, fetched 2026-10-27
