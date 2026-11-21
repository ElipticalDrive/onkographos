# Triplane and Planar Representations

Also called: Triplanar Representation, Triplane NeRF, Hybrid Triplane

## Overview
Triplane and Planar Representations model 3D scenes using a set of axis-aligned orthogonal 2D feature planes (typically $xy$, $xz$, and $yz$) to achieve full expressivity at scale while avoiding the cubic memory complexity of voxel-based representations [S1, S2]. This approach projects image features into a 3D volume and aggregates them into 2D planes, allowing for efficient rendering and optimization compared to dense 3D grids [S1].

## Shared principles
The core mechanism involves representing the scene geometry and appearance using a fixed number of 2D feature grids stored along canonical planes. Unlike the parent method's MLP-only approach, this representation explicitly stores features in a grid structure that is sampled via interpolation.

**Data Structure**
The scene is represented by a tensor $S$ (or $T$) of shape $\mathbb{R}^{3 \times C \times D \times D}$, where the first dimension indexes the three planes ($xy, xz, yz$), $C$ is the feature dimension, and $D \times D$ is the spatial resolution of each plane [S1, S2].

**Feature Aggregation**
To query a point $\mathbf{x} \in \mathbb{R}^3$, features are gathered by projecting $\mathbf{x}$ onto each of the canonical planes and bilinearly interpolating the nearby features [S2]. Let $\mathbf{f}_{xy}, \mathbf{f}_{xz}, \mathbf{f}_{yz}$ be the features gathered from the respective planes. These are combined, often via a mean operator $\Phi$, and passed to a lightweight MLP to predict radiance field properties (color, density, or other features) [S2].

**Rendering**
The gathered features condition the neural volumetric rendering process. Points are sampled along rays, and their features are aggregated to render images, similar to the ray sampling defined in the parent article [S1, S2].

## Subtopics

### [Image-Conditional Triplanar Representation](18169-image-conditional-triplanar-representation.md)
Image-Conditional Triplanar Representation is a 3D scene representation that models scenes using three axis-aligned orthogonal planes ($S = [S_{xy}, S_{xz}, S_{yz}]$) to achieve full expressivity at scale. It avoids the cubic memory complexity of voxel-based representations and the dimensionality limitations of 2D or Bird's Eye View (BEV) representations by projecting image features into a 3D volume and aggregating them into 2D planes.

**Inputs:** Input image $I \in \mathbb{R}^{H_i \times W_i \times 3}$, camera pose $\gamma_i$, camera intrinsic $K_i$.
**Outputs:** Triplanar feature tensor $S \in \mathbb{R}^{3 \times C \times D \times D}$, where the first dimension indexes the planes ($xy, xz, yz$), $C$ is the feature dimension, and $D \times D$ is the spatial resolution of each plane.

**Data Structures:**
1.  **Feature Map ($F_I$):** 2D tensor of shape $\mathbb{R}^{H_i/2 \times W_i/2 \times C}$.
2.  **3D Feature Volume ($V_F$):** Implicitly constructed grid where features are projected along rays.
3.  **Depth-Encoded Features ($V_Z$):** Output of the depth MLP.
4.  **Triplane ($S$):** Tensor $\mathbb{R}^{3 \times C \times D \times D}$.

**Steps:**
1.  **Image Encoding:** Extract low-resolution spatial features from the input image $I$ using an ImageNet pre-trained ConvNet backbone $E$. This transforms $I$ into a 2D feature map $F_I \in \mathbb{R}^{H_i/2 \times W_i/2 \times C}$.
2.  **Back-Projection:** Project local features from $F_I$ backwards along every ray to construct a 3D feature volume $V_F$. This projection uses the camera pose $\gamma_i$ and intrinsic $K_i$.
3.  **Depth Encoding:** Encode the depth of individual features in the grid using an MLP $V_Z = Z(V_F, x_c, d)$. The MLP takes as input:
    *   Concatenated features in the grid ($V_F$).
    *   Positions of the grid in the camera frame ($x_c$).
    *   Directions from the world frame positions ($x_w$) to the camera frame.
    The output is depth-encoded features $V_Z$.
4.  **Triplane Aggregation:** Obtain triplane features $S_{xy}, S_{xz}, S_{yz}$ by summing weighted depth-encoded features $V_Z$. The weights are softmax scores from aggregation MLPs $A_{xy}, A_{xz}, A_{yz}$ conditioned on $V_Z$ and world coordinates. The specific formulas are:
    $$
    S_{xy} = \sum_i w_{xy_i} V_{Z_i}, \quad w_{xy} = A_{xy}(V_{Z_i}, x_{w_z})
    $$
    $$
    S_{xz} = \sum_i w_{xz_i} V_{Z_i}, \quad w_{xz} = A_{xz}(V_{Z_j}, x_{w_y})
    $$
    $$
    S_{yz} = \sum_i w_{yz_i} V_{Z_i}, \quad w_{yz} = A_{yz}(V_{Z_j}, x_{w_x})
    $$
    where $w_{xy}, w_{xz}, w_{yz}$ are softmax scores obtained after summing over the $z, y,$ and $x$ dimensions respectively.
5.  **Upsampling and Convolution:** Transform the built image-conditional triplanes into a new $G$-channel output using learned 2D convolutions. The number of output channels is $G = C/4$. The spatial dimension of the planes is upsampled from $D \times D$ to the image feature space resolution ($H/2 \times W/2$).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def image_conditional_triplanar_representation(I, gamma_i, K_i, E, Z, A_xy, A_xz, A_yz, C, D):
    """
    I: Input image tensor (H_i, W_i, 3)
    gamma_i: Camera pose
    K_i: Camera intrinsic
    E: ImageNet pre-trained ConvNet backbone
    Z: Depth encoding MLP
    A_xy, A_xz, A_yz: Feature aggregation MLPs
    C: Feature dimension
    D: Spatial resolution of each plane
    """
    # Step 1: Image Encoding
    # E transforms I to 2D feature map F_I
    F_I = E(I) # Shape: (H_i/2, W_i/2, C)
    
    # Step 2: Back-Projection
    # Project features from F_I backwards along rays to 3D feature volume V_F
    # Using camera pose gamma_i and intrinsic K_i
    # Note: Implementation details of volumetric projection are assumed standard
    V_F = project_features_backwards(F_I, gamma_i, K_i)
    
    # Step 3: Depth Encoding
    # V_Z = Z(V_F, x_c, d)
    # x_c: positions of grid in camera frame
    # d: directions from world frame positions x_w to camera frame
    # For each point in the volume, compute x_c and d
    V_Z = Z(V_F, x_c, d)
    
    # Step 4: Triplane Aggregation
    # S_xy = sum_i w_xy_i * V_Z_i, where w_xy = softmax(A_xy(V_Z_i, x_w_z))
    # S_xz = sum_i w_xz_i * V_Z_i, where w_xz = softmax(A_xz(V_Z_j, x_w_y))
    # S_yz = sum_i w_yz_i * V_Z_i, where w_yz = softmax(A_yz(V_Z_j, x_w_x))
    
    # Compute weights for xy plane
    w_xy = softmax(A_xy(V_Z, x_w_z))
    S_xy = sum(w_xy * V_Z, dim=0) # Sum over feature dimensions
    
    # Compute weights for xz plane
    w_xz = softmax(A_xz(V_Z, x_w_y))
    S_xz = sum(w_xz * V_Z, dim=0)
    
    # Compute weights for yz plane
    w_yz = softmax(A_yz(V_Z, x_w_x))
    S_yz = sum(w_yz * V_Z, dim=0)
    
    # Stack into triplane S
    S = stack([S_xy, S_xz, S_yz]) # Shape: (3, C, D, D)
    
    # Step 5: Upsampling and Convolution
    # Upsample spatial dimension from D*D to H/2*W/2
    # Transform to G-channel output where G = C/4
    G = C // 4
    S_upsampled = upsample(S, target_size=(H_i/2, W_i/2))
    S_output = learned_2d_convolution(S_upsampled, out_channels=G)
    
    return S_output
```

**Edge Cases and Pitfalls:**
*   **Precision:** The back-projection and interpolation steps require careful handling of floating-point precision to avoid artifacts in the feature volume.
*   **Boundaries:** The projection relies on valid camera parameters; invalid poses or intrinsics may lead to incorrect feature placement.
*   **Degenerate Input:** If the input image lacks sufficient texture or detail, the extracted features $F_I$ may be insufficient for accurate 3D reconstruction.

### Triplane NeRF with Diffusion Prior
The framework uses a triplane NeRF representation where scene codes are optimized jointly with a diffusion prior to handle sparse-view data and ensure end-to-end coherence [S1].

**Parameter**
*   **Prediction Format:** v-parameterization $\hat{v}_\phi(x(t), t)$ from [43], such that $\hat{x} = \alpha(t)x(t) - \sigma(t)\hat{v}$ [S1].
*   **Weighting Function $w(t)$:** Uses SNR-based weighting $w(t) = (\alpha(t)/\sigma(t))^2\omega$ from Eq. (5), instead of LSGM mechanisms [S1].

**Layout**
*   **Scene code $x_i$:** Shape $\mathbb{R}^{3 \times 6 \times 128 \times 128}$, representing a triplane NeRF representation [S1].
*   **Denoising Network Input/Output:** Noisy and denoised triplane features, with channels of all three planes stacked together [S1].
*   **Denoising Network Architecture:** U-Net with 122M parameters [S1].

### EG3D Triplane Representation
EG3D learns unconditional 3D-aware image generation from a collection of single-view images and corresponding noisy camera poses, where each image has resolution $512 \times 512$ [S2].

**Step**
*   Render feature image $I_f \in \mathbb{R}^{32 \times 128 \times 128}$ and raw RGB image $I_{128} \in \mathbb{R}^{3 \times 128 \times 128}$ by accumulating points along rays and performing volume rendering [S2].
*   Extract dense depth map $I_D \in \mathbb{R}^{128 \times 128}$ from the volume rendering [S2].
*   Feed $I_f$ and $I_{128}$ into a 2D super-resolution network to yield final output $I_{512} \in \mathbb{R}^{3 \times 512 \times 512}$ [S2].

**Formula**
$$ \text{SuperRes}(I_f, I_{128}) = I_{512} $$
[S2]

**Parameter**
*   **Image resolution:** $512 \times 512$ for input images and final output [S2].

**Layout**
*   **Triplane representation $T$:** A tensor in $\mathbb{R}^{256 \times 256 \times 96}$, corresponding to 3 axis-aligned planes ($xy, xz, yz$), each with 32 channels and resolution $256 \times 256$ [S2].

**Number**
*   **Rendering speed:** 42 fps on RTX 3090 [S2].

### DoublePlane
On D-NeRF dataset, DoublePlane achieves PSNR 30.370, SSIM 0.961, LPIPS 0.054, Training Time 8m 04s [S3].

## Background
Triplane representations were introduced as an efficient alternative to dense voxel grids for 3D-aware image generation and NeRF optimization. The hybrid triplane representation was notably used in EG3D [S2] to enable fast rendering and high-quality 3D generation. Image-conditional triplanar representations were developed to generalize NeRF reconstruction from single or sparse views by conditioning the triplane features on input images [S1]. DoublePlane and HexPlane variants further optimized this structure for dynamic scenes [S3].

## Sources
[S1] Single-Stage Diffusion NeRF: A Unified Approach to 3D Generation and Reconstruction, https://doi.org/10.1109/iccv51070.2023.00229, fetched 2026-10-29
[S2] Real-Time Radiance Fields for Single-Image Portrait View Synthesis, https://doi.org/10.1145/3592460, fetched 2026-10-27
[S3] HexPlane: A Fast Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.00021, fetched 2026-10-26
