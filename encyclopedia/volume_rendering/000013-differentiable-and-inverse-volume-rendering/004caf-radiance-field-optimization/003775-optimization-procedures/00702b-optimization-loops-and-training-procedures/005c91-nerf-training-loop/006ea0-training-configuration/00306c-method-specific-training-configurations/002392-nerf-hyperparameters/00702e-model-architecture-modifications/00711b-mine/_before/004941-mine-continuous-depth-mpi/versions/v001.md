# MINE (Continuous Depth MPI)

Also called: Continuous Depth MPI, MINE: Towards Continuous Depth MPI with NeRF for Novel View Synthesis

## Overview
MINE (MINE: Towards Continuous Depth MPI with NeRF for Novel View Synthesis) is a method for novel view synthesis that reconstructs a scene using a continuous depth Multi-Plane Image (MPI) representation [S1]. It allows rendering of any target view via homography warping after a single encoder pass and a fixed number of decoder passes, independent of the number of target views [S1]. Unlike pixelNeRF and GRF, whose complexity scales linearly with the number of target views, MINE's complexity is constant with respect to the number of novel views, enabling dense supervision signals like SSIM and edge-aware smoothness loss during training [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single source image $I_{src}$ and camera intrinsics (focal lengths $f_x, f_y$, principal point $c_x, c_y$) [S1].
*   **Output:** A rendered image $\hat{I}_{tgt}$ and optionally a disparity map for a target view [S1].

**Data Structures:**
The method uses a Multi-Plane Image (MPI) representation consisting of $N_{\text{planes}}$ planes [S1]. Each plane $k$ (where $k \in \{1, \dots, N_{\text{planes}}\}$) contains:
*   Color $\mathbf{c}_k \in \mathbb{R}^{H \times W \times 3}$
*   Depth/Disparity $\sigma_k \in \mathbb{R}^{H \times W}$ (Note: The source uses $\sigma$ for the depth channel, distinct from NeRF's volume density, produced by taking the absolute value of the last channel of the decoder output) [S1].

**Number of Forward Passes:**
The total number of network forward passes $P$ is defined as [S1]:
$$
P_{\text{MINE}} = 1 + N_{\text{planes}}
$$
This contrasts with pixelNeRF ($P_{\text{pixelNeRF}} = 1 + N_{\text{targets}} \times N_{\text{points}} \times H \times W$) and GRF ($P_{\text{GRF}} = N_{\text{views}} + N_{\text{targets}} \times N_{\text{points}} \times H \times W$) [S1].

**Algorithm:**

1.  **Encoding:**
    Pass the source image $I_{src}$ through a ResNet50 encoder [S1]. Extract feature maps from five specific layers: `conv1`, `layer1`, `layer2`, `layer3`, and `layer4` [S1].

2.  **MPI Reconstruction (Decoding):**
    For each plane $k$ from $1$ to $N_{\text{planes}}$:
    a. Pass the encoder features through the depth decoder network [S1].
    b. The decoder outputs a 4-channel tensor $\mathbf{O}_k \in \mathbb{R}^{H \times W \times 4}$ [S1].
    c. Extract color $\mathbf{c}_k$ by applying a Sigmoid activation to the first three channels of $\mathbf{O}_k$ [S1].
    d. Extract depth/disparity $\sigma_k$ by taking the absolute value of the fourth channel of $\mathbf{O}_k$ [S1].
    e. Store $(\mathbf{c}_k, \sigma_k)$ as the $k$-th plane of the MPI [S1].

3.  **Rendering Target View:**
    a. Define the target camera intrinsics $(f_x^{tgt}, f_y^{tgt}, c_x^{tgt}, c_y^{tgt})$ [S1].
    b. For each pixel $(u, v)$ in the target image:
        i. Compute the ray corresponding to $(u, v)$ in the target view [S1].
        ii. Intersect the ray with the $N_{\text{planes}}$ planes to find intersection points [S1].
        iii. Perform homography warping on the reconstructed frustum (MPI) to map the source view features to the target view geometry [S1].
        iv. Composite the colors from the planes based on the depth ordering to produce $\hat{I}_{tgt}(u, v)$ [S1].
    *Note: The source states rendering is achieved by "applying homography warping to the reconstructed frustum" [S1]. Standard MPI rendering involves alpha-compositing from front to back. The source emphasizes the efficiency of this step being independent of $N_{\text{targets}}$ [S1].*

**Decoder Architecture Details:**
The decoder is based on the depth decoder from [12] with two additional downsampling blocks and two upsampling blocks [S1]. It takes encoder features and disparity encoding as input [S1]. The layers are defined as follows (Table 5 in [S1]):

| Layer | Type | Input Channels | Output Channels | Input Connection | Activation |
| :--- | :--- | :--- | :--- | :--- | :--- |
| downconv1 | Conv | 2048 | 512 | Encoder `layer4` | ELU |
| downconv2 | Conv | 512 | 256 | `downconv1` | ELU |
| upconv1_extra | Conv + Up | 256 | 256 | `downconv2` | ELU |
| upconv2_extra | Conv + Up | 256 | 2048 | `upconv1_extra` | ELU |
| upconv5 | Conv + Up | 2048 + 21 | 256 | cat(`upconv2_extra`, disparity encoding) | ELU |
| iconv5 | Conv | 256 + 1024 + 21 | 256 | cat(`upconv5`, encoder `layer3`, disparity encoding) | ELU |
| upconv4 | Conv + Up | 256 | 128 | `iconv5` | ELU |
| iconv4 | Conv | 128 + 512 + 21 | 128 | cat(`upconv4`, encoder `layer2`, disparity encoding) | ELU |
| output4 | Conv | 128 | 4 | `iconv4` | Sigmoid (RGB), abs ($\sigma$) |
| upconv3 | Conv + Up | 128 | 64 | `iconv4` | ELU |
| iconv3 | Conv | 64 + 256 + 21 | 64 | cat(`upconv3`, encoder `layer1`, disparity encoding) | ELU |
| output3 | Conv | 64 | 4 | `iconv3` | Sigmoid (RGB), abs ($\sigma$) |
| upconv2 | Conv + Up | 64 | 32 | `iconv3` | ELU |
| iconv2 | Conv | 32 + 64 + 21 | 32 | cat(`upconv2`, encoder `conv1`, disparity encoding) | ELU |
| output2 | Conv | 32 | 4 | `iconv2` | Sigmoid (RGB), abs ($\sigma$) |
| upconv1 | Conv + Up | 32 | 16 | `iconv2` | ELU |
| iconv1 | Conv | 16 | 16 | `upconv1` | ELU |
| output1 | Conv | 16 | 4 | `iconv1` | Sigmoid (RGB), abs ($\sigma$) |

*   **Upconv blocks:** Convolution, Batch Normalization, Activation, followed by 2x nearest neighbor upsampling [S1].
*   **Downconv blocks:** Max pooling (stride 2), Convolution, Activation [S1].
*   **Disparity Encoding:** Concatenated with features at specific levels (channels +21) [S1].

**Training Strategy:**
*   MINE renders the full target image and disparity map during training [S1].
*   **Loss Functions:**
    *   $L_{L1}$ and $L_{SSIM}$ are applied only to `output1` [S1].
    *   $L_{smooth}$ (edge-aware smoothness loss) is applied to all outputs (`output1`, `output2`, `output3`, `output4`) [S1].
*   **Viewing Direction:** MINE does not take viewing direction as input by default [S1]. It can be integrated by concatenating per-ray viewing directions with output feature maps at the target view (after warping) and using a lightweight fully convolutional network to predict view-dependent radiance [S1].

**Parameters:**
*   $N_{\text{planes}}$: Number of planes in the MPI representation. Not given in the sources.
*   **Flowers Light Fields Specifics:**
    *   Disparity range: $[3.0, 0.03]$ [S1].
    *   Grid distance (adjacent grids): $0.00128$ [S1].
    *   Camera Intrinsics (due to Lytro Illum camera shift):
        $$f_x = 0.868056, \quad f_y = 1.250000$$
        $$c_{xij} = 0.5 + 0.002667 \times i$$
        $$c_{yij} = 0.5 + 0.002667 \times j$$
        where $[i, j]$ is the index in the extracted $8 \times 8$ grid and $[0, 0]$ is the top-left view [S1].

**Edge Cases and Pitfalls:**
*   **Principal Point Shift:** In the Flowers Light Fields dataset, there is a shift in the principal point between different views due to the Lytro Illum camera [S1]. Since there is no metadata for the shift amount, it is set to a constant value based on the grid index $[i, j]$ [S1].
*   **View-Dependence:** The base method does not model view-dependent effects. If required, additional network passes are needed, increasing complexity but still remaining lower than pixelNeRF/GRF for many targets [S1].

## Operating characteristics
*   **Time Complexity:** The number of forward passes $P_{\text{MINE}} = 1 + N_{\text{planes}}$ is independent of the number of target views $N_{\text{targets}}$ [S1]. In contrast, pixelNeRF and GRF scale linearly with $N_{\text{targets}}$ [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:**
    *   Efficiency allows for dense supervision signals (SSIM, edge-aware smoothness) during training, which is infeasible for NeRF-like methods due to their inefficiency [S1].
    *   Generalizes well to large-scale real-world datasets (KITTI, NYU-V2, RealEstate10k, iBims-1, Flowers Light Fields) [S1].
*   **Limitations:**
    *   Does not natively support view-dependent radiance without additional architectural modifications (lightweight FCN) [S1].
    *   Requires careful handling of camera intrinsics shifts in specific datasets like Flowers Light Fields [S1].
*   **When to use:** When rendering many novel views from a single image is required, and dense training supervision is beneficial for generalization.
*   **When not to use:** When view-dependent effects are critical and the overhead of the additional lightweight FCN is prohibitive, or when the scene geometry is extremely complex and requires per-pixel ray marching (like standard NeRF) for higher fidelity, though MINE is verified on complex scenes like flowers [S1].

## Background
MINE was introduced in the paper "MINE: Towards Continuous Depth MPI with NeRF for Novel View Synthesis" [S1]. It was verified on datasets including KITTI, NYU-V2, RealEstate10k, Flowers Light Fields, and iBims-1 [S1]. The method builds on the encoder-decoder structure common in single-image novel view synthesis but replaces the per-pixel MLP inference with a fully convolutional decoder to reconstruct an MPI frustum [S1].

## Sources
[S1] MINE: Towards Continuous Depth MPI with NeRF for Novel View Synthesis, https://doi.org/10.1109/iccv48922.2021.01235, fetched 2026-10-27
