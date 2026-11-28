# Plane Sweep Volume (PSV)

Also called: Plane Sweep Volume (PSV) Averaging

## Overview
The Plane Sweep Volume (PSV) is a 5D tensor representation that encodes the epipolar geometry between a set of input views and a target view by projecting input images onto a series of depth planes [S1, unofficial]. It differs from standard volume representations by structuring data such that adjacent camera rays have adjacent encodings, allowing local operators to process epipolar lines efficiently [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $V$ input images $I_v$ with known camera intrinsics $K_v$ and extrinsics $(R_v, t_v)$ relative to a target view [S1, unofficial]. The target view has intrinsics $K_*$ and normal $n_*$ [S1, unofficial]. A set of $D$ depth planes is defined parallel to the target image plane, specified by distances $\{a_d\}_{d=1}^D$ from the target camera center [S1, unofficial].
*   **Output:** A 5D tensor $X \in \mathbb{R}^{D \times V \times 3 \times H \times W}$, where dimensions correspond to depth, view, color channels, height, and width [S1, unofficial].

**Data Structures**
The PSV tensor $X$ is indexed as $X_{d,v,c,h,w}$ [S1, unofficial].
*   $d \in \{1, \dots, D\}$: Depth plane index.
*   $v \in \{1, \dots, V\}$: Input view index.
*   $c \in \{1, 2, 3\}$: Color channel.
*   $h, w$: Pixel coordinates in the target image plane.

**Numbered Steps**
1.  **Define Depth Planes:** Establish $D$ planes parallel to the target image plane with normal $n_*$ [S1, unofficial]. The distance $a_d$ of each plane from the target camera center is chosen to cover the scene's depth range [S1, unofficial].
2.  **Compute Homographies:** For each input view $v$ and depth plane $d$, compute the homography matrix $H_{dv}$ that maps points from the target image plane to the input view plane at depth $a_d$ [S1, unofficial]. Assuming the world origin is at the target camera center ($R_* = I, t_* = 0$), the homography is:
    $$ H_{dv} = K_v (R_v - t_v n_*^T) K_*^{-1} $$
    where $K_v, K_*$ are intrinsic matrices, and $R_v, t_v$ are the rotation and translation of view $v$ relative to the target [S1, unofficial].
3.  **Project Images:** Apply the homography $H_{dv}$ to each input image $I_v$ to generate a projected image $X_{dv}$ for each depth plane [S1, unofficial].
4.  **Construct Tensor:** Assemble the projected images into the 5D tensor $X$ [S1, unofficial]. The slice $r_{hw}$ for a target pixel $(h,w)$ contains pixels sampled along the epipolar lines in all input views at all depths [S1, unofficial]. Specifically:
    $$ r_{hw} = \left\{ \left\{ X_{dv,c,h,w} \right\}_{c=1}^3 \right\}_{d=1}^D \right\}_{v=1}^V $$
    This structure ensures that operations along the depth dimension correspond to operations along individual epipolar lines, and operations along the view dimension correspond to operations between corresponding epipolar lines [S1, unofficial].

**Parameters**
Typical values for $D$, $V$, $H$, and $W$ are Not given in the sources.

**Edge Cases and Pitfalls**
*   **Depth Coverage:** The distances $\{a_d\}$ must be chosen such that the scene of interest is adequately covered [S1, unofficial].
*   **Memory:** The 5D structure requires significant memory, scaling as $O(D \cdot V \cdot H \cdot W)$ [S1, unofficial].

**Averaging Variants**
*   **Mean PSV:** Average the tensor $X$ over the view dimension $V$ to obtain a tensor of shape $\mathbb{R}^{D \times 3 \times H \times W}$ [S1, unofficial]. This highlights epipolar geometry: parts of objects lying on a specific depth plane appear in focus, while others appear blurry [S1, unofficial].
*   **Blurry Estimate:** Average $X$ over both view and depth dimensions to produce a blurry estimate of the target view [S1, unofficial].

## Operating characteristics
*   **Memory Complexity:** $O(D \cdot V \cdot H \cdot W)$ [S1, unofficial]. The tensor size grows linearly with the number of depth planes, views, and image resolution [S1, unofficial].
*   **Strengths:** Encodes epipolar geometry explicitly, allowing efficient local processing via convolutions [S1, unofficial]. Adjacent rays have adjacent encodings, facilitating the use of simple local operators [S1, unofficial].
*   **Limitations:** High memory consumption for large $D$ and $V$ [S1, unofficial]. Requires known camera intrinsics/extrinsics and depth bounds covering the scene [S1, unofficial].
*   **When to Use:** Novel view synthesis where generalizable models are needed, and epipolar consistency is important [S1, unofficial].

## Background
The Plane Sweep Volume was introduced in the context of generalizable novel view synthesis, specifically in the Global Latent Neural Rendering framework [S1, unofficial]. The concept of averaging the PSV to highlight depth planes is closely related to the original plane sweep algorithm by Collins for depth estimation [S1, unofficial]. The PSV generalizes the 4D tensor (single view) to a 5D tensor (multiple views) to leverage multi-view epipolar geometry [S1, unofficial].

## Sources
[S1] Global Latent Neural Rendering, https://cvpr.thecvf.com/virtual/2024/poster/30744, fetched 2026-10-30 (unofficial)
