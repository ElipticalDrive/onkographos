# Encoder-Decoder Training

Also called: Reconstruction Framework

## Overview
Encoder-Decoder Training is a method for training a pair of encoder $\theta_{enc}$ and decoder $\theta_{dec}$ to reconstruct images from latent codes embedded in a 3D representation, such as the RNR-Map [S1]. It resembles an autoencoder but is adapted for 3D environments where the neural radiance fields are conditioned on these latent codes [S1]. Unlike standard NeRF optimization which directly optimizes voxel or coordinate-based fields, this approach generalizes to unseen environments by learning a mapping from 2D image features to 3D latent codes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An RGBD image $I_t$ and its relative pose $\Delta p_t = (\Delta x_t, \Delta y_t, \Delta \alpha_t)$ from the previous pose [S1].
*   **Output:** Reconstructed images at query poses [S1].

**Data Structures:**
*   **RNR-Map ($m$):** A grid-form map containing latent codes [S1].
*   **Latent Code ($C_t$):** A feature map of size $H \times W \times D$, where $H, W$ are image height/width and $D$ is the channel size [S1].
*   **Encoder Parameters ($\theta_{enc}$):** Network parameters for the encoder [S1].
*   **Decoder Parameters ($\theta_{dec}$):** Network parameters for the decoder [S1].

**Steps:**
1.  **Encode:** The encoder $\theta_{enc}$ encodes the input RGBD image $I_t$ into a same-sized feature map $C_t \in \mathbb{R}^{H \times W \times D}$ [S1].
2.  **Register (Embed):** The latent codes are embedded into the RNR-Map $m$ via a registration function $F_{reg}$ [S1]. This involves:
    *   Mapping each pixel feature $c_{h,w} \in C_t$ to a 3D world position $[q_x, q_y, q_z]^T$ using inverse camera projection based on camera intrinsic $K$, extrinsic matrix $[R|t]$, and depth $d_{h,w}$ [S1].
    *   Aggregating features into grid cells. For a grid cell $(u,v)$, the set of features $X_{(u,v)}$ is defined as:
        $$ X_{(u,v)} = \left\{ c_{h,w} \in C_t \mid u = \left\lfloor \frac{q_x(h,w)}{s} \right\rceil, v = \left\lfloor \frac{q_y(h,w)}{s} \right\rceil \right\} $$
        where $s$ is the grid scale [S1].
    *   The value $m(u,v)$ is the mean of features in $X_{(u,v)}$:
        $$ m(u,v) = \frac{1}{n(u,v)} \sum_{c_i \in X_{(u,v)}} c_i, \quad n(u,v) = |X_{(u,v)}| $$
        where $n(u,v)$ is the count of features in the cell [S1].
3.  **Decode:** At a query pose, the decoder $\theta_{dec}$ samples latent codes from the RNR-Map $m$ along each camera ray corresponding to each pixel and renders the corresponding images [S1].
4.  **Update:** The parameters $\theta_{enc}$ and $\theta_{dec}$ are updated to minimize the L1 loss between the rendered image and the target image [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encoder_decoder_training_step(I_t, p_t, K, R_t, t_t, depth_t, s, theta_enc, theta_dec, m):
    """
    I_t: RGBD image tensor [H, W, 4]
    p_t: Relative pose [delta_x, delta_y, delta_alpha]
    K: Camera intrinsic matrix
    R_t, t_t: Camera extrinsic rotation and translation
    depth_t: Depth map [H, W]
    s: Grid scale for RNR-Map
    theta_enc: Encoder network parameters
    theta_dec: Decoder network parameters
    m: RNR-Map grid structure
    """
    # Step 1: Encode image to latent features
    C_t = theta_enc(I_t) # Output shape: [H, W, D]
    
    # Step 2: Register features into RNR-Map
    # Calculate 3D world positions for each pixel
    # q_xyz = inverse_projection(K, R_t, t_t, depth_t)
    # This step aggregates C_t into grid cells m
    # For each pixel (h,w) with feature c_h_w and world pos q_xyz:
    #   u = round(q_xyz.x / s)
    #   v = round(q_xyz.y / s)
    #   Add c_h_w to bucket X_(u,v)
    # m(u,v) = mean(X_(u,v))
    m = register_features(C_t, q_xyz, s)
    
    # Step 3: Decode latent codes to render image at query pose
    # Sample m along rays defined by query pose
    rendered_image = theta_dec(m, query_pose)
    
    # Step 4: Compute Loss and Update
    # Assuming target_image is available for supervised training
    loss = L1_loss(rendered_image, target_image)
    update_parameters(theta_enc, theta_dec, loss)
    
    return rendered_image, m
```

**Parameters:**
*   **Grid Scale ($s$):** Determines the resolution of the RNR-Map grid [S1].
*   **Loss Function:** L1 loss [S1].

**Edge Cases and Pitfalls:**
*   **Empty Grid Cells:** If no features map to a grid cell $(u,v)$, $n(u,v)=0$. The formula assumes $n(u,v) > 0$ for the mean calculation [S1].
*   **Pose Accumulation:** The agent determines its absolute pose $p_t$ by cumulating relative pose differences $\Delta p_t$ from a start pose $p_0 = (0,0,0)$ [S1]. Errors in relative pose estimation can lead to misalignment in the RNR-Map.

## Subtopics

### [Decode-Scene](17506-decode-scene.md)
A method where the NeRF is trained in latent space but supervised by a photometric loss computed in the RGB space after decoding the rendered latent images. It uses the decoder and supervises the pipeline with RGB images while keeping the NeRF in 3D latent space. The loss is defined as:
$$ \min_{\alpha, T} L_D(\alpha, T) \triangleq \mathbb{E}_p \| x_p - D(R_\alpha(T, p)) \| $$
where $D$ represents the decoder, $R_\alpha$ is the rendering operation in latent space, and $x_p$ is the target RGB image. This ensures high-fidelity color output supervision.

### Encode-Decode-Scene
A hybrid training method that uses a weighted combination of the Encode-Scene loss ($L_E$) and Decode-Scene loss ($L_D$) to train the latent NeRF, ensuring both latent consistency and high-fidelity RGB reconstruction [S2, unofficial]. It utilizes both the encoder and decoder modules of a 3D-aware autoencoder [S2, unofficial]. The steps are: 1. Encode input images to latent space. 2. Train Tri-Planes in latent space. 3. Decode rendered latent images. 4. Compute loss in RGB space or latent space, or a combination [S2, unofficial]. The optimization objective is:
$$ \min_{\alpha, T} (1-t)L_D(\alpha, T) + tL_E(\alpha, T) $$
where $t \in (0, 1)$ is a mixing coefficient [S2, unofficial]. This method combines the accelerated training of Encode-Scene with the high-fidelity color output supervision of Decode-Scene [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is designed to be real-time capable, avoiding the significant time required for inference in other NeRF-based navigation methods that require rendering-based optimization [S1].
*   **Memory Complexity:** Depends on the size of the RNR-Map grid and the dimensionality of the latent codes $D$ [S1].
*   **Strengths:** Enables generalization to unseen environments using pretrained encoder and decoder [S1]. Real-time capability for visual navigation [S1].
*   **Limitations:** Autoencoding a 3D environment is not trivial compared to 2D images [S1]. Requires accurate pose estimation for registration [S1].

## Background
The method was proposed in the context of visual navigation using a Renderable Neural Radiance Map (RNR-Map) [S1]. It addresses the challenge of generalizing NeRF-based methods to unseen environments by treating the plenoptic function as a decoder and scene codes as latent codes with Gaussian priors, building on the NeRF as an Auto-Decoder framework [S1, Parent]. The RNR-Map is designed to contain the overall visual information of a 3D environment in a grid form [S1].

## Sources
[S1] Renderable Neural Radiance Map for Visual Navigation, https://doi.org/10.1109/cvpr52729.2023.00878, fetched 2026-10-26
[S2] Exploring 3D-aware Latent Spaces for Efficiently Learning Numerous Scenes, https://doi.org/10.48550/arXiv.2403.11678, fetched 2026-10-29 (unofficial)
