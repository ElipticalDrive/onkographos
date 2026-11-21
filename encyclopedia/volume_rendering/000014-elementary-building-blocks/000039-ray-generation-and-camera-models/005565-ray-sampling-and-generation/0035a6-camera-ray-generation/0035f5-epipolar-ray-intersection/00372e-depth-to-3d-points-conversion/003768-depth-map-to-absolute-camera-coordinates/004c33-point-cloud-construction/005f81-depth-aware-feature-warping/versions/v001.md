# Depth-aware Feature Warping


## Overview
Depth-aware Feature Warping projects 2D features from a source view to a target view using estimated depth and camera poses, enabling multi-view consistency checks or feature aggregation for depth estimation [S1, unofficial]. It differs from standard geometric warping by explicitly handling depth uncertainty through a depth volume, allowing the method to aggregate features across multiple depth hypotheses rather than a single depth plane.

## How it works
**Inputs:**
- Source feature map $F_s \in \mathbb{R}^{B \times C \times H \times W}$, where $B$ is batch size, $C$ is channels, $H$ is height, $W$ is width.
- Source camera intrinsics $K \in \mathbb{R}^{B \times 3 \times 3}$.
- Relative pose from source to target view $P \in \mathbb{R}^{B \times 4 \times 4}$, composed of rotation $R \in \mathbb{R}^{B \times 3 \times 3}$ and translation $t \in \mathbb{R}^{B \times 3 \times 1}$.
- Depth volume $D \in \mathbb{R}^{B \times D \times H \times W}$, where $D$ is the number of depth candidates.

**Outputs:**
- Warped feature map $F_w \in \mathbb{R}^{B \times C \times D \times H \times W}$ (or flattened spatial dimensions depending on implementation).

**Data Structures:**
- Homogeneous pixel coordinate grid: Shape $[B, 3, H, W]$. The first two channels are normalized coordinates $(x, y)$, and the third channel is all ones.
- 3D points in camera space: Shape $[B, 3, H \times W]$.
- Transformed 3D points: Shape $[B, 3, D, H \times W]$.
- Sampling grid: Shape $[B, D \times H, W, 2]$ or $[B, D, H, W, 2]$ depending on `grid_sample` requirements.

**Parameters:**
- `clamp_min_depth`: Minimum depth value for clamping $z$ during reprojection to prevent division by zero. Default is $10^{-3}$ [S1, unofficial].

**Steps:**

1.  **Generate Pixel Grid:** Create a homogeneous grid of pixel coordinates. For each pixel $(u, v)$ in the source image, construct a vector $[u, v, 1]^T$. This results in a tensor of shape $[B, 3, H, W]$ [S1, unofficial].

2.  **Back-project to 3D Camera Space:** Flatten the spatial dimensions of the grid to $[B, 3, H \times W]$. Multiply by the inverse of the source camera intrinsics $K^{-1}$ to obtain 3D points in the source camera coordinate system (normalized by depth, effectively rays):
    $$ P_{cam} = K^{-1} \cdot G_{flat} $$
    Resulting shape: $[B, 3, H \times W]$ [S1, unofficial].

3.  **Transform to Target Coordinate System:**
    a. Apply the rotation $R$ from the relative pose $P$:
    $$ P_{rot} = R \cdot P_{cam} $$
    b. Repeat the rotated points along the depth dimension to match the depth volume shape $[B, 3, D, H \times W]$.
    c. Scale by the depth values $d_i$ from the depth volume $D$:
    $$ P_{scaled} = P_{rot} \cdot D_{reshaped} $$
    where $D_{reshaped}$ has shape $[B, 1, D, H \times W]$ [S1, unofficial].

4.  **Translate by Pose Translation:** Add the translation vector $t$ from the relative pose to the scaled points:
    $$ P_{target} = P_{scaled} + t $$
    Resulting shape: $[B, 3, D, H \times W]$ [S1, unofficial].

5.  **Reproject to Target Image Plane:** Multiply the 3D points in the target coordinate system by the target camera intrinsics $K_{target}$ (often assumed same as source or provided separately). Reshape points to $[B, 3, D \times H \times W]$ before multiplication:
    $$ P_{proj} = K_{target} \cdot P_{target, flat} $$
    Reshape back to $[B, 3, D, H \times W]$ [S1, unofficial].

6.  **Compute Normalized Pixel Coordinates:** Divide the $x$ and $y$ coordinates by the $z$ coordinate (depth in target view). Clamp $z$ to `clamp_min_depth` to avoid division by zero:
    $$ u_{norm} = \frac{P_{proj, x}}{\max(P_{proj, z}, \text{clamp\_min\_depth})} $$
    $$ v_{norm} = \frac{P_{proj, y}}{\max(P_{proj, z}, \text{clamp\_min\_depth})} $$
    Resulting shape for $u_{norm}, v_{norm}$: $[B, D, H \times W]$ [S1, unofficial].

7.  **Normalize to [-1, 1] Range:** Convert normalized pixel coordinates to the range $[-1, 1]$ required by bilinear sampling functions:
    $$ x_{grid} = \frac{2 u_{norm}}{W - 1} - 1 $$
    $$ y_{grid} = \frac{2 v_{norm}}{H - 1} - 1 $$
    Resulting shape: $[B, D, H \times W]$ [S1, unofficial].

8.  **Construct Sampling Grid:** Stack $x_{grid}$ and $y_{grid}$ to form a sampling grid. The grid must be shaped appropriately for the interpolation function, typically $[B, D \times H, W, 2]$ or $[B, D, H, W, 2]$ with channels last [S1, unofficial].

9.  **Sample Features:** Use bilinear interpolation (`grid_sample`) to sample features from the source feature map $F_s$ using the computed grid. Use zero padding for out-of-bounds pixels:
    $$ F_w = \text{grid\_sample}(F_s, \text{Grid}, \text{mode}=\text{"bilinear"}, \text{padding\_mode}=\text{"zeros"}, \text{align\_corners}=\text{True}) $$
    Resulting shape: $[B, C, D, H, W]$ [S1, unofficial].

**Pitfalls:**
- Large feature maps (e.g., $>1,000,000$ elements) can cause CUDNN errors in `grid_sample`. In such cases, disable CUDNN for the sampling operation [S1, unofficial].
- Division by zero must be handled via clamping $z$ [S1, unofficial].
- Coordinate system conventions (e.g., origin at center vs. corner, axis directions) must be consistent between back-projection and reprojection [G].

## Operating characteristics
- **Time Complexity:** $O(B \cdot D \cdot H \cdot W)$, dominated by the 3D point transformation and bilinear sampling.
- **Memory Complexity:** $O(B \cdot D \cdot H \cdot W)$ for storing the transformed 3D points and sampling grids.
- **Strengths:** Enables robust feature aggregation across views by accounting for depth uncertainty. Allows for multi-view consistency checks by warping features from multiple source views to a target view.
- **Limitations:** Computationally expensive for high-resolution images and large depth volumes. Sensitive to errors in depth estimation and camera pose accuracy.
- **When to use:** Multi-view stereo depth estimation, novel view synthesis, and 3D reconstruction tasks requiring feature alignment across views.
- **When not to use:** Real-time applications with strict latency constraints and limited depth uncertainty.

## Background
Depth-aware Feature Warping is a fundamental technique in multi-view geometry and computer vision, particularly in deep learning-based multi-view stereo (MVS) and novel view synthesis. It allows neural networks to leverage information from multiple viewpoints by aligning features in a common coordinate system. The method is widely used in frameworks like DepthSplat for efficient depth estimation and feature aggregation [S1, unofficial].

## Sources
[S1] cvg/depthsplat, https://github.com/cvg/depthsplat, fetched 2026-10-27 (unofficial)
