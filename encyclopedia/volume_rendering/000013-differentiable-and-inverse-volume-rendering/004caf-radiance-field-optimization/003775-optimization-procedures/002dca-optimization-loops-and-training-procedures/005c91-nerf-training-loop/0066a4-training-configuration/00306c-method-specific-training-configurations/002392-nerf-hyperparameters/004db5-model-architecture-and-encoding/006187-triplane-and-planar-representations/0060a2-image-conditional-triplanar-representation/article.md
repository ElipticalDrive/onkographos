# Image-Conditional Triplanar Representation

Also called: Tri-plane Features

## Overview
Image-Conditional Triplanar Representation is a 3D scene representation that models scenes using three axis-aligned orthogonal planes ($S = [S_{xy}, S_{xz}, S_{yz}]$) to achieve full expressivity at scale [S1]. It avoids the cubic memory complexity of voxel-based representations and the dimensionality limitations of 2D or Bird's Eye View (BEV) representations by projecting image features into a 3D volume and aggregating them into 2D planes [S1].

## How it works
**Inputs:** Input image $I \in \mathbb{R}^{H_i \times W_i \times 3}$, camera pose $\gamma_i$, camera intrinsic $K_i$.
**Outputs:** Triplanar feature tensor $S \in \mathbb{R}^{3 \times C \times D \times D}$, where the first dimension indexes the planes ($xy, xz, yz$), $C$ is the feature dimension, and $D \times D$ is the spatial resolution of each plane [S1].

**Data Structures:**
1.  **Feature Map ($F_I$):** 2D tensor of shape $\mathbb{R}^{H_i/2 \times W_i/2 \times C}$ [S1].
2.  **3D Feature Volume ($V_F$):** Implicitly constructed grid where features are projected along rays [S1].
3.  **Depth-Encoded Features ($V_Z$):** Output of the depth MLP [S1].
4.  **Triplane ($S$):** Tensor $\mathbb{R}^{3 \times C \times D \times D}$ [S1].

**Steps:**
1.  **Image Encoding:** Extract low-resolution spatial features from the input image $I$ using an ImageNet pre-trained ConvNet backbone $E$. This transforms $I$ into a 2D feature map $F_I \in \mathbb{R}^{H_i/2 \times W_i/2 \times C}$ [S1].
2.  **Back-Projection:** Project local features from $F_I$ backwards along every ray to construct a 3D feature volume $V_F$. This projection uses the camera pose $\gamma_i$ and intrinsic $K_i$ [S1].
3.  **Depth Encoding:** Encode the depth of individual features in the grid using an MLP $V_Z = Z(V_F, x_c, d)$. The MLP takes as input:
    *   Concatenated features in the grid ($V_F$).
    *   Positions of the grid in the camera frame ($x_c$).
    *   Directions from the world frame positions ($x_w$) to the camera frame [S1].
    The output is depth-encoded features $V_Z$ [S1].
4.  **Triplane Aggregation:** Obtain triplane features $S_{xy}, S_{xz}, S_{yz}$ by summing weighted depth-encoded features $V_Z$. The weights are softmax scores from aggregation MLPs $A_{xy}, A_{xz}, A_{yz}$ conditioned on $V_Z$ and world coordinates [S1]. The specific formulas are:
    $$
    S_{xy} = \sum_i w_{xy_i} V_{Z_i}, \quad w_{xy} = A_{xy}(V_{Z_i}, x_{w_z})
    $$
    $$
    S_{xz} = \sum_i w_{xz_i} V_{Z_i}, \quad w_{xz} = A_{xz}(V_{Z_j}, x_{w_y})
    $$
    $$
    S_{yz} = \sum_i w_{yz_i} V_{Z_i}, \quad w_{yz} = A_{yz}(V_{Z_j}, x_{w_x})
    $$
    where $w_{xy}, w_{xz}, w_{yz}$ are softmax scores obtained after summing over the $z, y,$ and $x$ dimensions respectively [S1].
5.  **Upsampling and Convolution:** Transform the built image-conditional triplanes into a new $G$-channel output using learned 2D convolutions. The number of output channels is $G = C/4$. The spatial dimension of the planes is upsampled from $D \times D$ to the image feature space resolution ($H/2 \times W/2$) [S1].

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
*   **Precision:** The back-projection and interpolation steps require careful handling of floating-point precision to avoid artifacts in the feature volume [S1].
*   **Boundaries:** The projection relies on valid camera parameters; invalid poses or intrinsics may lead to incorrect feature placement [S1].
*   **Degenerate Input:** If the input image lacks sufficient texture or detail, the extracted features $F_I$ may be insufficient for accurate 3D reconstruction [S1].

## Operating characteristics
*   **Time Complexity:** Avoids the cubic complexity of 3D CNNs and voxel-based representations [S1].
*   **Memory Complexity:** Linear in the number of planes and their resolution, avoiding cubic scaling with volume size [S1].
*   **Expressivity:** Full expressivity at scale, modeling all three dimensions unlike 2D or BEV representations [S1].
*   **Strengths:** Computationally efficient compared to 3D CNNs; more expressive than 2D/BEV methods; acts as a global feature representation [S1].
*   **Limitations:** Relies on the quality of the initial image features and camera parameters [S1].
*   **When to Use:** Sparse view synthesis of outdoor scenes where computational efficiency and expressivity are critical [S1].
*   **When Not to Use:** When high-resolution volumetric details are required that cannot be captured by planar projections [S1].

## Background
The Image-Conditional Triplanar Representation was introduced in "NeO 360: Neural Fields for Sparse View Synthesis of Outdoor Scenes" [S1]. It builds on prior works involving volumetric reconstruction [S1]. The method is designed to address the limitations of voxel-based and 2D/BEV representations in modeling 3D scenes [S1].

## Sources
[S1] NeO 360: Neural Fields for Sparse View Synthesis of Outdoor Scenes, https://doi.org/10.1109/iccv51070.2023.00843, fetched 2026-10-26
