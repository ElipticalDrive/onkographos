# Iterative Alignment Strategy

Also called: Misalignment Correction via Optical Flow

## Overview
The Iterative Alignment Strategy is a post-processing and data-generation technique used to mitigate reconstruction artifacts in Neural Radiance Fields (NeRF) caused by inaccurate camera poses and non-rigid scene motion [S1]. It addresses the "domain gap" where NeRF-rendered views are blurry due to misalignment, violating the sharp-pair assumption of standard optical flow estimators [S1]. Unlike direct alignment methods that optimize a loss function during training, this strategy iteratively warps ground truth images using optical flow to create a better-aligned training set, leveraging the sharpening effect of improved alignment to refine flow estimates in subsequent rounds [S1].

## How it works
The method operates as an iterative loop to generate aligned training and testing data [S1]. It assumes access to a NeRF model (e.g., mip-NeRF 360) and a set of input images with potentially erroneous camera poses [S1].

**Inputs:**
*   Current NeRF model parameters (MLPs) [S1].
*   Original ground truth images $I_{gt}$ [S1].
*   Original camera poses $P$ [S1].
*   Optical flow estimator (PWC-Net) [S1].

**Outputs:**
*   Warped ground truth images $I_{warped}$ [S1].
*   Updated NeRF model parameters [S1].

**Data Structures:**
*   Images: 2D arrays of pixel values (RGB) [S1].
*   Optical Flow: 2D vector fields $(u, v)$ representing displacement from source to target [S1].

**Steps:**
1.  **Render:** Render images $I_{render}$ from the current NeRF model using the original camera poses $P$ [S1].
2.  **Estimate Flow:** Calculate the optical flow from the rendered views $I_{render}$ to the ground truth views $I_{gt}$ using PWC-Net [S1].
    *   *Note:* The rendered images are often blurry due to misalignment, which violates the general optical flow assumption of sharp paired inputs. However, PWC-Net produces reasonable results based on global structures and shapes [S1].
3.  **Warp:** Warp the ground truth images $I_{gt}$ using the estimated optical flows to build a new, better-aligned training set $I_{warped}$ [S1]. The warping aligns the ground truth content with the geometry estimated by the NeRF [S1].
4.  **Retrain:** Train the NeRF model using the new aligned training set $I_{warped}$ [S1].
5.  **Iterate:** Repeat steps 1-4 for 2-3 iterations [S1]. Each iteration leverages sharper images (from the improved NeRF) to improve flow accuracy, which in turn enables better alignment for the next round [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iterative_alignment_strategy(nerf_model, gt_images, poses, num_iterations=3):
    """Iteratively aligns ground truth images to NeRF geometry using optical flow [S1]."""
    current_images = gt_images
    for _ in range(num_iterations):
        # Step 1: Render current NeRF model using original poses
        rendered_images = [nerf_model.render(p) for p in poses]
        # Step 2: Calculate optical flow from rendered to ground truth using PWC-Net [S1]
        # Note: PWC-Net handles blurry inputs by relying on global structures [S1]
        flow_fields = [estimate_flow(rendered, gt) for rendered, gt in zip(rendered_images, current_images)]
        # Step 3: Warp ground truth images using estimated flows to build aligned training set [S1]
        aligned_images = [warp(gt, f) for gt, f in zip(current_images, flow_fields)]
        # Step 4: Retrain NeRF on aligned images (implicit for next iteration)
        # nerf_model.train(aligned_images, poses)
        current_images = aligned_images
    return current_images

def estimate_flow(source, target):
    # Compute optical flow field F from source to target using PWC-Net [S1]
    # Returns 2D vector field of displacements
    pass  # Implementation details of PWC-Net are external [S1]

def warp(image, flow):
    # Sample image pixels at coordinates displaced by flow field using bilinear interpolation
    # Aligns ground truth content with NeRF geometry [S1]
    pass  # Standard image warping operation
```

**Parameters:**
*   **Iterations:** 2-3 iterations are typically sufficient to reach best quality [S1].
*   **Optical Flow Network:** PWC-Net [S1]. RAFT was considered but rejected due to memory constraints on high-resolution images [S1].

**Edge Cases and Pitfalls:**
*   **Memory Constraints:** PWC-Net may run out of memory on high-resolution images. Alternative flow networks like RAFT were rejected for the same reason [S1].
*   **Blurry Inputs:** NeRF rendered images are often blurry due to misalignment, violating the sharp-pair assumption of optical flow estimators. This can reduce flow accuracy, though global structures help mitigate this [S1].
*   **Non-Rigid Motion:** The strategy addresses misalignment from camera pose errors and non-rigid motion [S1].

## Subtopics

### [Non-Rigid Refinement](11817-non-rigid-refinement.md)
Non-Rigid Refinement is a post-processing alignment step that uses an iterative optical flow network to correct non-rigid deformations or misalignments in NeRF features by warping neighbor features to test features. It addresses the domain gap between blurry rendered novel views and sharp neighboring images, which causes standard 3D alignment to fail due to camera pose errors and geometry estimation inaccuracies. Unlike general-purpose alignment, this method is trained end-to-end specifically for the task of refining feature correspondences in the presence of these domain gaps.

### Direct Alignment (RGB Alignment)
Direct Alignment is a training technique that applies a direct alignment loss between rendered images and input RGB images to improve camera pose estimation and structure reconstruction in neural radiance fields [S2, unofficial]. For every frame, it selects a surrogate view from a neighborhood starting from 2 frames till 5 frames on either side of the keyframe to perform direct alignment [S2, unofficial]. It is trained for 100k iterations [S2, unofficial]. Unlike methods that require expansive optic-flow depth or foundation models for regularization, direct alignment avoids local minima in joint camera pose estimation without them [S2, unofficial]. However, violations of photo-consistency can hamper reconstructions, particularly with semi-transparent objects [S2, unofficial]. It has been applied to LLFF and DTU datasets, showing average boosts of 60% in rotation and 53% in translation error reduction [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method requires multiple iterations of NeRF training and optical flow estimation. Each iteration involves rendering all training views, running PWC-Net, and warping images. The total time scales with the number of iterations (typically 2-3) multiplied by the cost of one NeRF training epoch plus flow estimation [S1].
*   **Memory Complexity:** High memory usage due to the need to store and process high-resolution images and optical flow fields. PWC-Net may run out of memory on high-resolution images, and RAFT was rejected for the same reason [S1].
*   **Performance:** The strategy improves PSNR by +0.62dB on the warped test set for large MLPs (4096 channels) and +0.23dB for smaller models (1024 channels) [S1]. It also improves LPIPS and SSIM metrics [S1].
*   **Strengths:** Effectively reduces artifacts caused by misalignment; leverages the self-improving nature of NeRF rendering (sharper images lead to better flow) [S1].
*   **Limitations:** Does not represent the core proposed algorithm of the AligNeRF paper (which is a loss function); serves primarily for analysis and data generation [S1]. Relies on the quality of the optical flow estimator; fails if the initial NeRF is too poor to provide useful global structures for flow estimation [S1].
*   **When to Use:** When camera poses are inaccurate and cause significant blurring in NeRF renders, and when computational resources allow for iterative retraining and flow estimation [S1].
*   **When Not to Use:** When memory is constrained for high-resolution images; when the scene has severe non-rigid motion that cannot be captured by global optical flow [S1].

## Background
The Iterative Alignment Strategy was introduced in the AligNeRF paper [S1]. It was developed to address the issue of misalignment in NeRF training, particularly in the context of the mip-NeRF 360 dataset [S1]. The authors observed that standard NeRF training with inaccurate poses leads to blurry renders, which in turn hampers the accuracy of optical flow estimation if used for alignment [S1]. The iterative approach was proposed to break this cycle by progressively aligning the training data [S1]. The strategy was used for analysis and data generation in the experiments, rather than as the primary algorithm of the work, which focused on an alignment-aware training loss [S1].

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] ACCV 2024 Open Access Repository, https://openaccess.thecvf.com/content/ACCV2024/html/Garg_Direct_Alignment_for_Robust_NeRF_Learning_ACCV_2024_paper.html, fetched 2026-10-29 (unofficial)
