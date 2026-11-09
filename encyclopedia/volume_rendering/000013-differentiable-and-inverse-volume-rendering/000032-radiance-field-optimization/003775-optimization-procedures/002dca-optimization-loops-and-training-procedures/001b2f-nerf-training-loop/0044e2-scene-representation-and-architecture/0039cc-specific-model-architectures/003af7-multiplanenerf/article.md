# MultiPlaneNeRF

Also called: MultiPlaneNeRF

## Overview
MultiPlaneNeRF is a radiance field representation that replaces the trainable volumetric storage of vanilla NeRF with a non-trainable set of 2D images [S1]. It solves the problems of long training times and poor generalization to unseen data inherent in vanilla NeRF by projecting 3D points onto these fixed 2D planes to create an aggregate feature vector, which is then processed by a small implicit decoder [S1]. Unlike vanilla NeRF, which encodes object-specific shape and color in network weights requiring separate training per object, MultiPlaneNeRF allows a single decoder to generalize across many objects by simply swapping the input 2D image set [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $n$ 2D training images $\{I_1, \dots, I_n\}$ with known camera poses (intrinsic and extrinsic parameters). A 3D point coordinate $\mathbf{x} = (x, y, z)$.
*   **Output:** Volume density $\sigma \in \mathbb{R}$ and RGB color $\mathbf{c} \in \mathbb{R}^3$.

**Data Structures:**
*   **2D Representation:** A list of $n$ images $I_j \in \mathbb{R}^{H \times W \times C_{img}}$, where $H, W$ are height and width, and $C_{img}$ is the number of channels (typically 3 for RGB). These images are static and non-trainable.
*   **Implicit Decoder:** A small Multi-Layer Perceptron (MLP) $F_\Theta$. The source does not specify the exact layer dimensions or activation functions, only describing it as "very shallow" or "small" [S1].

**Algorithm:**

1.  **Data Splitting:** Split the initial set of $n$ 2D training images into two subsets:
    *   $S_{rep}$: Images used to build the 2D representation (input to the decoder).
    *   $S_{train}$: Images used as the training set to calculate the weights $\Theta$ of the implicit decoder [S1].
    *   *Note:* In generalization experiments, the model is trained on a dataset (e.g., ShapeNet) where each object has 50 images of size $200 \times 200$ with transparent backgrounds from random camera positions [S1].

2.  **Projection and Aggregation:** For a given 3D point $\mathbf{x} = (x, y, z)$ and a specific 2D image $I_j$ from the representation set $S_{rep}$:
    *   Project $\mathbf{x}$ onto the 2D image plane of $I_j$ using the known camera pose of $I_j$.
    *   Sample the pixel value $\mathbf{p}_j$ at the projected coordinates from $I_j$.
    *   Repeat for all $k$ images in the representation subset.
    *   Concatenate the sampled pixel values to form the aggregate information vector $\mathbf{Z}(\mathbf{x}) \in \mathbb{R}^{5k}$.
    *   *Formula:* $\mathbf{Z}(\mathbf{x}) = [\mathbf{p}_1; \mathbf{p}_2; \dots; \mathbf{p}_k]$. The dimension is specified as $\mathbb{R}^{5k}$, where $k$ relates to the number of images or channels [S1]. This implies each image contributes 5 values to the vector (possibly RGB + depth or other channels, though the source explicitly states the output dimension is $5k$).

3.  **Decoding:** Pass the aggregate vector $\mathbf{Z}(\mathbf{x})$ through the implicit decoder $F_\Theta$:
    *   $\sigma, \mathbf{c} = F_\Theta(\mathbf{Z}(\mathbf{x}))$.
    *   The decoder predicts the volume density $\sigma$ and the RGB color $\mathbf{c}$ [S1].

4.  **Rendering:** Use the predicted $\sigma$ and $\mathbf{c}$ in the standard volumetric rendering equation along a ray $\mathbf{r}$:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\delta_i$ is the distance between samples [Parent].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def project_point_to_image(point_3d, camera_pose, image_shape):
    """
    Projects a 3D point onto a 2D image given camera pose.
    Returns the pixel value at the projected location.
    Note: Exact projection math (intrinsic/extrinsic) is assumed to be handled
    by standard computer vision libraries or defined by the camera model.
    """
    # 1. Transform 3D point to camera coordinates using extrinsic matrix
    point_cam = camera_pose.extrinsic @ np.append(point_3d, 1)
    
    # 2. Project to 2D using intrinsic matrix
    point_2d_hom = camera_pose.intrinsic @ point_cam[:3]
    point_2d = point_2d_hom[:2] / point_2d_hom[2]
    
    # 3. Sample pixel value from image
    # Assuming bilinear interpolation or nearest neighbor
    x, y = int(point_2d[0]), int(point_2d[1])
    if 0 <= x < image_shape[1] and 0 <= y < image_shape[0]:
        return image[y, x] # Returns RGB vector
    else:
        return np.zeros(3) # Default value for out-of-bounds

def aggregate_representation(point_3d, image_set):
    """
    Projects 3D point onto all images in the set and aggregates features.
    Returns vector Z of size 5k.
    """
    features = []
    for img in image_set:
        # img contains the image data and its camera pose
        pixel_val = project_point_to_image(point_3d, img.pose, img.shape)
        
        # The source specifies the output dimension is R^{5k}.
        # If pixel_val is RGB (3 channels), we need 2 more values per image.
        # The source does not specify what these 2 extra values are.
        # We assume they might be depth, alpha, or padding to reach 5k.
        # For implementation, we concatenate RGB and assume 2 dummy/derived values.
        # [G] Placeholder for the 2 additional channels required to reach dimension 5.
        # In practice, this might involve depth maps or normal vectors if available,
        # or simply repeating/zero-padding if not specified.
        # Here we assume the input images provide 5 channels or we derive 2.
        # Since the source says "R^{5k}", and standard images are 3 channels,
        # we must account for the discrepancy.
        
        # Let's assume the image provides 5 channels (e.g., RGB + Depth + Alpha)
        # or we pad. The source is ambiguous on the source of the 5 channels.
        # We will assume the image data itself provides the necessary 5 values.
        # If only RGB is available, this step requires an assumption.
        # [G] Assuming the image set provides 5-channel data or we append zeros.
        feature_vec = np.concatenate([pixel_val, np.zeros(2)]) 
        features.append(feature_vec)
        
    return np.concatenate(features)

def multi_plane_nerf_decode(point_3d, image_set, decoder_mlp):
    """
    Main inference function.
    """
    Z = aggregate_representation(point_3d, image_set)
    sigma, color = decoder_mlp(Z)
    return sigma, color

def render_ray(ray_origin, ray_direction, image_set, decoder_mlp, num_samples):
    """
    Renders a single ray.
    """
    # Sample points along the ray
    t_vals = np.linspace(0.1, 10.0, num_samples) # [G] Typical near/far bounds
    points_3d = ray_origin + ray_direction * t_vals[:, np.newaxis]
    
    sigmas = []
    colors = []
    
    for p in points_3d:
        sigma, color = multi_plane_nerf_decode(p, image_set, decoder_mlp)
        sigmas.append(sigma)
        colors.append(color)
        
    sigmas = np.array(sigmas)
    colors = np.array(colors)
    
    # Volumetric Rendering
    deltas = np.diff(t_vals, append=10.0) # [G] Approximation for last delta
    alpha = 1.0 - np.exp(-sigmas * deltas)
    T = np.cumprod(np.concatenate(([1.0], 1.0 - alpha[:-1])), axis=0)
    
    contribution = alpha * T * colors
    final_color = np.sum(contribution, axis=0)
    
    return final_color
```

**Parameters:**
*   $k$: Number of images in the representation subset.
*   $n$: Total number of training images.
*   Image Resolution: $200 \times 200$ pixels (typical in experiments) [S1].
*   Decoder Size: "Small" or "very shallow" [S1]. Exact dimensions not given.

**Edge Cases and Pitfalls:**
*   **Generalization vs. Quality:** Training on a large dataset results in slightly worse rendering quality compared to training each object separately [S1].
*   **Projection Boundaries:** If a 3D point projects outside the image bounds, the sampling must handle this (e.g., zero padding or clamping) [G].
*   **Dimension Mismatch:** The source specifies an output dimension of $\mathbb{R}^{5k}$ but standard RGB images are 3 channels. The source does not explain the origin of the extra 2 channels per image. This is a critical implementation gap [S1].

## Operating characteristics
*   **Time Complexity:** Training time is significantly reduced compared to vanilla NeRF because the representation (2D images) is non-trainable and the decoder is small [S1]. Inference time is also reduced due to the shallow decoder [S1].
*   **Memory Complexity:** Memory usage is dominated by the storage of the 2D images ($n \times H \times W \times C$) and the small MLP weights. This is generally lower than storing a large voxel grid or a large MLP encoding the entire scene [S1].
*   **Strengths:**
    *   Generalizes to unseen objects by simply changing the input 2D images without retraining the decoder [S1].
    *   Achieves comparable results to state-of-the-art models (vanilla NeRF, voxel-based NSFV) with fewer parameters [S1].
    *   Can be used as a component in generative models like GANs (e.g., MultiPlaneGAN) [S1].
*   **Limitations:**
    *   Slightly worse rendering quality when trained on large datasets compared to per-object training [S1].
    *   Requires known camera poses for the input images [S1].
    *   Interpolation between objects is possible by mixing images, but the quality of this transition depends on the similarity of the underlying shapes [S1].

## Background
MultiPlaneNeRF was proposed to address the limitations of vanilla NeRF, specifically long training times and poor generalization to unseen data [S1]. It was introduced in the paper "MultiPlaneNeRF: Neural radiance field with non-trainable representation" [S1]. The method builds on the idea of using non-trainable representations (existing images) and a small implicit decoder, similar in spirit to TriPlane concepts but using arbitrary 2D images rather than fixed planes [S1]. It has been used in the context of MultiPlaneGAN, a GAN-based generative model [S1].

## Sources
[S1] MultiPlaneNeRF: Neural radiance field with non-trainable representation, https://doi.org/10.1016/j.eswa.2025.127350, fetched 2026-10-29
