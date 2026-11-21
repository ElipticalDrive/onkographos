# MultiPlaneNeRF

Also called: MultiPlane NeRF

## Overview
MultiPlaneNeRF is a radiance field representation that replaces trainable 3D parameters (such as voxel grids or implicit weights) with a fixed set of $n$ pre-existing 2D images [S1]. It solves the problems of long training times and poor generalization to unseen data found in vanilla NeRF by projecting 3D points onto these 2D planes to create a non-trainable aggregate input vector [S1]. Unlike siblings like [Mega-NeRF Spatial Partitioning](8776-mega-nerf-spatial-partitioning.md) which use spatial hashing, or [Quadri-Plane Radiance Fields](11128-quadri-plane-radiance-fields.md) which use learnable feature planes, MultiPlaneNeRF uses static, real-world (or synthetic) images as the sole 3D representation, allowing a small implicit decoder to generalize across many objects by simply swapping the input images [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $n$ 2D training images $I_1, \dots, I_n$, each of resolution $N \times N \times 3$ [S1]. A 3D point $\mathbf{x} = (x, y, z)$ and a viewing direction $\mathbf{d}$ [S1]. For generalization tasks, the camera positions $P(I_i)$ for each image are also provided [S1].
*   **Outputs:** Volume density $\sigma$ and RGB color $\mathbf{c}$ [S1].

**Data Structures:**
*   **Images:** Stored as part of the model architecture. They are non-trainable and must be stored with the model weights [S1].
*   **Aggregate Vector $Z$:** A vector of dimension $R^{5n}$ for standard rendering, or $R^{8n}$ for generalization tasks [S1].

**Algorithm:**
1.  **Projection:** For a given 3D point $\mathbf{x}$ and each image $I_i$, compute the 2D projected coordinates $z_{I_i}$ by projecting $\mathbf{x}$ onto the image plane [S1].
    $$
    \text{Pr}(\mathbf{x}, I_i) = z_{I_i} \in \mathbb{R}^2
    $$
    This uses the standard NeRF camera-to-world transformation reversed to obtain world-to-camera coordinates [S1].

2.  **Color Sampling:** Sample the RGB color $\mathbf{c}_{I_i}$ from image $I_i$ at coordinates $z_{I_i}$. Since the projected coordinates $z_{I_i}$ generally do not align with pixel centers, use linear interpolation to obtain the color [S1].
    $$
    \mathbf{c}_{I_i} = I_i[z_{I_i}] \in \mathbb{R}^3
    $$

3.  **Aggregate Vector Construction:** Concatenate the sampled colors, the projected 2D coordinates, and (for generalization) the camera positions to form the input vector $Z$ for the decoder [S1].
    *   **Standard Rendering:**
        $$
        Z(\mathbf{x}) = [\mathbf{c}_{I_1}, z_{I_1}, \dots, \mathbf{c}_{I_n}, z_{I_n}] \in \mathbb{R}^{5n}
        $$
    *   **Generalization Rendering:**
        $$
        Z(\mathbf{x}) = [\mathbf{c}_{I_1}, z_{I_1}, P(I_1), \dots, \mathbf{c}_{I_n}, z_{I_n}, P(I_n)] \in \mathbb{R}^{8n}
        $$
        where $P(I_i)$ is the position of the camera dedicated to image $I_i$ [S1].

4.  **Implicit Decoding:** Pass $Z(\mathbf{x})$ to the implicit decoder network $F_{\text{MultiPlaneNeRF}}$ along with the viewing direction $\mathbf{d}$ to predict density $\sigma$ and color $\mathbf{c}$ [S1].
    $$
    F_{\text{MultiPlaneNeRF}}(\mathbf{x}, \mathbf{d}, I_1, \dots, I_n; \Theta) = (\mathbf{c}, \sigma)
    $$
    For generalization:
    $$
    F_{\text{MultiPlaneNeRF}}(\mathbf{x}, \mathbf{d}; \Theta, I_1, P(I_1), \dots, I_n, P(I_n)) = (\mathbf{c}, \sigma)
    $$
    The decoder architecture is similar to vanilla NeRF, using approximately 0.5M parameters [S1].

5.  **Rendering:** Compute the pixel color $C(\mathbf{r})$ using the standard volumetric rendering integral [S1]:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def project_point_to_image(x_3d, camera_matrix):
    """
    Projects a 3D point x_3d (3,) onto the 2D image plane using the camera matrix.
    Returns 2D coordinates (2,).
    Note: The exact projection function depends on the camera model (e.g., pinhole).
    This is a placeholder for the standard NeRF world-to-camera projection.
    """
    # Implementation detail: Apply inverse camera matrix to x_3d, then perspective divide.
    # Not given in the sources, but standard for NeRF.
    pass

def sample_image_color(image, uv_coords):
    """
    Samples RGB color from image (H, W, 3) at normalized uv_coords (2,).
    Uses linear interpolation.
    """
    # Implementation detail: Bilinear interpolation of image at uv_coords.
    # Not given in the sources, but standard for image sampling.
    pass

def build_aggregate_vector(x_3d, images, camera_positions=None):
    """
    Builds the aggregate input vector Z for the MultiPlaneNeRF decoder.
    
    Args:
        x_3d: 3D point (3,)
        images: List of n images, each (H, W, 3)
        camera_positions: Optional list of n camera positions (3,) for generalization
        
    Returns:
        Z: Aggregate vector (5n,) or (8n,)
    """
    n = len(images)
    components = []
    
    for i in range(n):
        I_i = images[i]
        
        # 1. Project 3D point to 2D image coordinates
        # Assuming camera_positions[i] provides the necessary camera intrinsics/extrinsics
        # to perform the projection.
        z_Ii = project_point_to_image(x_3d, camera_positions[i])
        
        # 2. Sample RGB color at projected coordinates
        c_Ii = sample_image_color(I_i, z_Ii)
        
        # 3. Concatenate color and 2D coordinates
        # c_Ii is (3,), z_Ii is (2,) -> combined is (5,)
        components.append(np.concatenate([c_Ii, z_Ii]))
        
        # 4. If generalization mode, add camera position
        if camera_positions is not None:
            P_Ii = camera_positions[i]
            # P_Ii is (3,) -> combined with previous (5,) is (8,)
            # Reconstructing the full vector for this plane
            components[-1] = np.concatenate([c_Ii, z_Ii, P_Ii])
            
    Z = np.concatenate(components)
    return Z

def multi_plane_nerf_forward(x_3d, d, images, camera_positions=None, decoder_weights=None):
    """
    Forward pass of MultiPlaneNeRF.
    
    Args:
        x_3d: 3D point (3,)
        d: Viewing direction (3,)
        images: List of n images
        camera_positions: Optional list of n camera positions
        decoder_weights: Weights for the implicit decoder
        
    Returns:
        sigma: Volume density (scalar)
        c: RGB color (3,)
    """
    # Build aggregate vector
    Z = build_aggregate_vector(x_3d, images, camera_positions)
    
    # Decode
    # The decoder takes Z and d as input.
    # Architecture: Similar to vanilla NeRF MLP.
    # Not given in the sources, but standard for NeRF decoders.
    sigma, c = decoder_forward(Z, d, decoder_weights)
    
    return sigma, c
```

**Parameters:**
*   $n$: Number of images in the representation. Increasing $n$ improves results but increases memory and computation [S1].
*   $N$: Resolution of the input images ($N \times N \times 3$) [S1].
*   Decoder Parameters $\Theta$: Approximately 0.5M parameters [S1].

**Edge Cases and Pitfalls:**
*   **Training on Large Datasets:** Training on a large dataset results in slightly worse rendering quality compared to training each object separately [S1].
*   **Generalization:** Camera positions $P(I_i)$ are only added to the input for generalization tasks; they do not change rendering results for single-object training [S1].
*   **Interpolation:** The model can interpolate between objects by mixing images from two different objects (using $k$ images from the first and $n-k$ from the second) [S1].

## Subtopics

### MultiPlaneGAN
A GAN-based generative model that uses the MultiPlaneNeRF decoder as a component, replacing the TriPlane decoder in EG3D [S1]. It produces interpretable representations since the planes are 2D images with three RGB channels [S1].

### Generalization Training
Training the decoder on a large dataset (e.g., ShapeNet) to generalize across many objects and classes, allowing the model to render unseen objects by simply changing the input images without retraining the decoder [S1].

## Operating characteristics
*   **Time and Memory Complexity:** The model uses fewer parameters than other hybrid methods (like voxel-based NSVF) because the 3D representation is non-trainable and stored as images [S1]. The decoder is small (~0.5M parameters) [S1].
*   **Measured Performance:** MultiPlaneNeRF achieves results comparable to state-of-the-art models (like vanilla NeRF and NSVF) on synthesizing new views [S1]. It obtains slightly better results than auto-encoder architectures (Points2NeRF) on test sets [S1].
*   **Scaling:** Results improve when the number of images in the representation increases [S1]. Better results are also obtained when the resolution of training images matches the expected image resolution [S1].
*   **Strengths:** Non-trainable 3D representation allows for fast training of the decoder and generalization to unseen objects by swapping images [S1].
*   **Limitations:** Trade-off between rendering quality and generalization; training on a large dataset yields slightly worse quality than training each object separately [S1].

## Background
MultiPlaneNeRF was introduced to address the limitations of vanilla NeRF, specifically long training times and poor generalization to unseen data [S1]. It was proposed as a method that works directly on 2D images, projecting 3D points to produce non-trainable representations [S1]. The model was evaluated on classical rendering tasks (Diffuse Synthetic 360, Realistic Synthetic 360) and generalization tasks (ShapeNet base dataset) [S1]. It has been used as a component in generative models like MultiPlaneGAN, an analog of EG3D GAN [S1].

## Sources
[S1] MultiPlaneNeRF: Neural radiance field with non-trainable representation, https://doi.org/10.1016/j.eswa.2025.127350, fetched 2026-10-29
