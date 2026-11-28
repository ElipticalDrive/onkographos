# Canonical Frame Estimation via Facial Landmarks

Also called: Camera Alignment via Procrustes, Facial Landmark-based Canonical Frame

## Overview
This method estimates a canonical coordinate frame for selfie and full-body captures using facial landmarks, enabling the automatic generation of novel view trajectories for NeRF training [S1]. It serves as a preprocessing step that aligns all input camera poses to a canonical coordinate system using 3D facial keypoints and similarity transformation [S2]. Unlike general Structure-from-Motion (SfM) alignment which relies on scene geometry, this method anchors the coordinate system to specific semantic features of the subject (the face), ensuring consistent orientation regardless of the initial camera pose [S1, S2].

## How it works
**Inputs:**
1. A set of $N$ images with associated camera poses $P_i \in \mathbb{R}^{4 \times 4}$ (extrinsics) and intrinsics, obtained from SfM [S1, S2].
2. 2D facial landmark detections for each image [S1].

**Outputs:**
1. A canonical coordinate frame defined by a rotation matrix $R_{can}$, translation vector $t_{can}$, and scale factor $s$ [S1, S2].
2. Transformed camera poses $P'_i$ aligned to this canonical frame [S2].

**Data Structures:**
*   **2D Landmarks:** For each image $i$, a set of $K$ 2D points $l_{i,k} = (u_{i,k}, v_{i,k})$ where $k \in \{1, \dots, K\}$. MediaPipe’s face mesh is used to extract these [S1].
*   **3D Keypoints:** A set of $M=5$ 3D points $X_k = (x_k, y_k, z_k)$ representing the outer eye corners, nose, mouth, and chin [S2].
*   **Canonical Keypoints:** A fixed set of 5 reference points $X^{ref}_k$ computed as the median location of these keypoints across the first 260 subjects in the training set [S2].

**Steps:**

1.  **2D Landmark Detection:** For each input image $i$, compute 2D facial landmarks $l_{i,k}$ using MediaPipe’s face mesh [S1].
2.  **3D Triangulation:** Triangulate the 2D landmarks into 3D space using the SfM camera poses $P_i$ and intrinsic parameters [S1]. Let the resulting 3D points for image $i$ be $\hat{X}_{i,k}$.
    *   *Note:* This step assumes the subject is static. If the subject moves, the triangulation is technically incorrect but observed to be sufficiently good for defining the frame even with head rotation [S1].
3.  **Keypoint Selection:** From the triangulated points, identify the 5 specific keypoints corresponding to the outer eye corners, nose, mouth, and chin [S2]. Let the set of 3D points for image $i$ be $X_i = \{\hat{X}_{i,1}, \dots, \hat{X}_{i,5}\}$.
4.  **Similarity Transform Calculation:** Calculate a similarity transform (rotation $R$, translation $t$, scale $s$) that maps the subject's 3D keypoints $X_i$ to the canonical reference keypoints $X^{ref}$ using Procrustes analysis [S2].
    *   The canonical frame is centered at the face mesh [S1].
    *   The orientation is defined as: $+y$ up, $+x$ right, $-z$ into the face [S1].
    *   The scale is set such that the distance between the eyes (interpupillary distance) matches 6 cm [S1].
5.  **Pose Transformation:** Apply the computed similarity transform to all camera poses $P_i$ to obtain the canonical poses $P'_i$ [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def estimate_canonical_frame(image_poses, intrinsics, image_paths):
    """
    Estimates a canonical frame and transforms camera poses.
    
    Args:
        image_poses: List of 4x4 numpy arrays (extrinsics) from SfM.
        intrinsics: Camera intrinsic matrix K.
        image_paths: List of image file paths.
        
    Returns:
        canonical_transform: 4x4 similarity transform matrix.
        transformed_poses: List of transformed 4x4 pose matrices.
    """
    # Step 1: Detect 2D landmarks and Triangulate to 3D
    # Note: In practice, this requires a face detection library (e.g., MediaPipe)
    # and a triangulation function. We assume get_2d_landmarks and triangulate_3d
    # are available helpers.
    
    all_3d_landmarks = []
    for i, img_path in enumerate(image_paths):
        # Get 2D landmarks for image i
        landmarks_2d = get_2d_landmarks(img_path) # Returns Nx2 array
        
        # Triangulate using the i-th camera pose and intrinsics
        # This projects 2D points back to 3D rays and intersects them
        # across multiple views or uses the known pose if multi-view is not 
        # explicitly handled in this simplified loop. 
        # The source implies triangulation using SfM poses [S1].
        landmarks_3d = triangulate_3d(landmarks_2d, image_poses[i], intrinsics)
        all_3d_landmarks.append(landmarks_3d)
        
    # Step 2: Select 5 specific keypoints
    # Indices depend on MediaPipe face mesh ordering. 
    # We assume indices for: left eye outer, right eye outer, nose tip, mouth center, chin.
    keypoint_indices = [33, 263, 1, 61, 152] 
    subject_keypoints = []
    
    for landmarks_3d in all_3d_landmarks:
        kp = landmarks_3d[keypoint_indices]
        subject_keypoints.append(kp)
        
    # Step 3: Define Canonical Reference Keypoints
    # Computed as median across first 260 subjects [S2]
    # For implementation, these are pre-computed constants.
    canonical_keypoints = get_canonical_reference_keypoints() 
    
    # Step 4: Calculate Similarity Transform via Procrustes
    # We align the subject's keypoints to the canonical keypoints.
    # Since the subject may be in different poses, we align the set of keypoints
    # from a representative frame (or average if available) to the canonical frame.
    # The source [S2] implies a single transform per subject/session.
    
    # Using the first frame's keypoints as the representative subject geometry
    X_subject = subject_keypoints[0]
    
    # Procrustes analysis to find R, t, s such that:
    # s * R * X_subject + t = X_canonical
    R, t, s = procrustes_analysis(X_subject, canonical_keypoints)
    
    # Step 5: Enforce Metric Scale (Interpupillary Distance = 6 cm)
    # The Procrustes solution provides relative scale. We adjust s to match 6cm.
    current_ipd = np.linalg.norm(X_subject[1] - X_subject[0]) # Distance between eyes
    target_ipd = 0.06 # 6 cm in meters
    s = s * (target_ipd / current_ipd)
    
    # Construct the 4x4 similarity transform matrix
    # T = [ s*R  t ]
    #     [  0   1 ]
    canonical_transform = np.eye(4)
    canonical_transform[:3, :3] = s * R
    canonical_transform[:3, 3] = t
    
    # Step 6: Transform all camera poses
    transformed_poses = []
    for pose in image_poses:
        # The camera pose P transforms world to camera.
        # To align the world (subject) to canonical, we apply the inverse of the 
        # subject-to-canonical transform to the world coordinates.
        # Alternatively, if P is [R_cam|t_cam], the new pose P' is:
        # P' = T_canonical * P
        # This assumes P is in the original SfM world frame.
        new_pose = canonical_transform @ pose
        transformed_poses.append(new_pose)
        
    return canonical_transform, transformed_poses

def procrustes_analysis(X, Y):
    """
    Computes the similarity transform (R, t, s) to map X to Y.
    X, Y are (M, 3) arrays.
    """
    # Center points
    X_mean = np.mean(X, axis=0)
    Y_mean = np.mean(Y, axis=0)
    X_c = X - X_mean
    Y_c = Y - Y_mean
    
    # Compute scale
    s = np.sum(Y_c**2) / np.sum(X_c**2)
    s = np.sqrt(s)
    
    # Compute rotation using SVD
    H = X_c.T @ Y_c
    U, S, Vt = np.linalg.svd(H)
    R = Vt.T @ U.T
    
    # Ensure proper rotation (det(R) = 1)
    if np.linalg.det(R) < 0:
        Vt[-1, :] *= -1
        R = Vt.T @ U.T
        
    # Compute translation
    t = Y_mean - s * R @ X_mean
    
    return R, t, s
```

**Parameters:**
*   **Interpupillary Distance:** 6 cm [S1].
*   **Canonical Keypoints:** Median location of 5 keypoints (outer eye corners, nose, mouth, chin) across the first 260 subjects in the training set [S2].
*   **Orientation:** $+y$ up, $+x$ right, $-z$ into the face [S1].

**Edge Cases and Pitfalls:**
*   **Subject Motion:** 3D triangulation of facial landmarks is only strictly correct if the subject is static [S1]. However, it is observed to be sufficiently good for defining the coordinate frame even when the subject rotates their head side-to-side [S1].
*   **Missing Segmentation:** This step is skipped for captures where segmentation masks cannot be obtained (e.g., BADMINTON and B ROOM datasets in the original study) [S1].
*   **Animal Captures:** For animal captures, manual virtual camera paths are generated instead of using this automatic method [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by 2D landmark detection (MediaPipe) and 3D triangulation. Procrustes analysis is negligible ($O(M)$ for $M=5$ keypoints).
*   **Memory Complexity:** Low. Requires storing 2D landmarks and 5 3D keypoints per image.
*   **Strengths:** Provides a consistent, semantically meaningful canonical frame for human subjects, enabling automatic novel view trajectory generation (e.g., figure-eight paths) [S1]. It is robust to initial camera pose variations [S2].
*   **Limitations:** Relies on the accuracy of facial landmark detection and SfM poses. Performance degrades if the subject moves significantly during capture, though head rotation is tolerated [S1]. Not applicable to non-human subjects without manual intervention [S1].

## Background
This method was introduced in the context of NeRFies, a framework for reconstructing and animating humans from video [S1]. It addresses the challenge of camera registration in the presence of a moving foreground by anchoring the coordinate system to the subject's face rather than the static background [S1]. The use of Procrustes analysis for alignment is a standard technique in computer vision for shape matching [S2]. The canonical keypoints are derived from a large dataset of 1450 volunteers, with the reference frame computed from the first 260 subjects [S2].

## Sources
[S1] Nerfies: Deformable Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00581, fetched 2026-10-26
[S2] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
