# Replica Dataset Preparation

Also called: Replica Data Processing

## Overview
Replica Dataset Preparation filters and corrects the synthetic Replica indoor scan dataset for neural rendering training and evaluation [S1, S2]. It addresses unrealistic camera poses (e.g., inside furniture), label ambiguity, and coordinate system misalignment by filtering invalid renders, normalizing poses to the origin, and semi-automatically correcting semantic labels [S1, S2].

## How it works
**Inputs:** RGB images, camera poses, semantic labels, and point clouds from the Replica dataset [S1, S2].
**Outputs:** Filtered image sets, corrected semantic labels, normalized camera poses, and near/far clipping parameters [S1, S2].

### Data Structures
The input data consists of:
- `Images`: RGB arrays of resolution $340 \times 600$ [S2].
- `Poses`: Camera poses for the rendered images [S1, S2].
- `Labels`: Semantic segmentation maps aligned with images [S1].
- `Point Clouds`: 3D coordinates associated with the scene [S1].

### Steps
1. **Scene Selection:** Select specific scenes from the Replica dataset. Standard preparation uses `room_0`, `room_1`, `office_3`, and `office_4` [S1]. For sparse-view variants, use `room0`, `room1`, `room2`, `office0`, `office1`, `office2`, and `office3` [S2].
2. **Geometric Filtering:** Render images from the provided camera poses. Filter out any images where the camera position is inside furniture or other geometry, as these are unrealistically rendered [S1].
3. **Coordinate Normalization:** Adjust the camera poses so that the 3D points visible in the selected training images are centered around the world origin $(0,0,0)$ [S2]. This is critical for pose refinement stability [S2].
4. **Clipping Plane Setup:** Set NeRF near/far clipping planes. Default is $(0.1, 10.0)$ [S1]. For `office_3`, use $(0.1, 15.0)$ [S1].
5. **Semantic Label Correction (Semi-Automatic):**
   - Apply LSeg to predict semantic label maps for each RGB image [S1].
   - Evaluate the accuracy of LSeg predictions against the original Replica label map [S1].
   - Ignore points with labels having accuracy almost or exactly zero from evaluation/training [S1].
   - Manually fix label ambiguities by ignoring specific labels or merging them (e.g., merge 'rug' and 'floor') [S1].
6. **Sparse-View Sub-sampling (Variant):**
   - Sub-sample every $k$-th frame from the video sequence [S2].
   - Adapt $k$ per scene to ensure a minimum of 20% covisible regions between any selected view and its neighbors [S2].
   - Randomly select a triplet of consecutive training images from the sub-sampled set [S2].

### Parameters
- **Image Resolution:** $340 \times 600$ pixels [S2].
- **Near/Far Clipping:** $(0.1, 10.0)$ generally; $(0.1, 15.0)$ for `office_3` [S1].
- **Covisibility Threshold:** Minimum 20% overlap between selected views [S2].

### Edge Cases
- **Collapsed Geometry:** Replica data suffers from collapsed geometry and less photo-realistic appearances compared to real scans [S1].
- **Label Ambiguity:** Objects may have bad appearance or intrinsic indistinguishability (e.g., rug vs. floor) [S1].
- **Unobserved Regions:** Training images may not cover all scene regions; evaluation must account for generalization to unobserved areas via propagation [S1].

## Subtopics

### Replica Dataset Sparse-View Preparation
A variant designed for sparse-view reconstruction tasks [S2]. It uses 7 scenes (`room0`, `room1`, `room2`, `office0`, `office1`, `office2`, `office3`) [S2]. It sub-samples every $k$-th frame from the original video (1500–3000 frames per scene) to create wide-baseline scenarios [S2]. The sampling rate $k$ is adapted per scene to ensure at least 20% covisible regions between selected views [S2]. It randomly selects a triplet of consecutive training images from these sub-sampled frames [S2]. Poses are adjusted to center the scene around the origin to benefit pose refinement [S2].

## Operating characteristics
- **Data Quality:** The source dataset is moderate-quality, suffering from collapsed geometry, less photo-realistic appearances, and defective annotations [S1].
- **Pose Refinement:** Centering the scene around the origin significantly benefits pose refinement stability [S2].
- **Evaluation Scope:** Training images may not cover all regions; evaluation measures generalization to unobserved regions via propagation [S1].
- **Limitations:** Label quality requires manual intervention to resolve ambiguities like 'rug' vs 'floor' [S1].

## Background
The Replica dataset is a moderate-quality indoor reconstruction dataset with semantic segmentation labels, originally created for proof-of-concept work in neural rendering [S1]. It is based on synthetic scans with randomly generated pose trajectories [S1]. The preparation method was introduced to address issues like camera-in-geometry poses and label inaccuracies [S1]. The sparse-view variant was developed to evaluate methods under realistic sparse-input conditions [S2].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S2] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
