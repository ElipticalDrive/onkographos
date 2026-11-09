# MultiPlaneNeRF

Also called: MultiPlaneNeRF

## Overview
MultiPlaneNeRF is a radiance field representation that replaces the trainable volumetric encoding of vanilla NeRF with a non-trainable 2D image-based projection [S1]. It solves the problems of long training times and poor generalization to unseen data by projecting 3D points onto a fixed set of 2D training images to create an aggregate feature vector, which is then processed by a small, shallow implicit decoder [S1]. Unlike vanilla NeRF, which encodes object-specific shape and color in network weights requiring separate training per object, MultiPlaneNeRF allows a single decoder to generalize across many objects by simply swapping the input 2D image representation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $n$ 2D training images $\{I_1, \dots, I_n\}$ with known camera poses; a 3D query point $\mathbf{x} = (x, y, z)$; and a viewing direction $\mathbf{d}$.
*   **Output:** Volume density $\sigma \in \mathbb{R}$ and RGB color $\mathbf{c} \in \mathbb{R}^3$.

**Data Structures:**
*   **2D Representation:** A collection of $n$ 2D images. In the reported experiments, these are $200 \times 200$ images with transparent backgrounds [S1].
*   **Aggregate Information Vector:** $Z(\mathbf{x}) \in \mathbb{R}^{5k}$, where $k$ is a parameter related to the number of images or channels used in the projection [S1]. The exact construction of $k$ is not specified beyond its relation to the image set size [S1].
*   **Implicit Decoder:** A small Multi-Layer Perceptron (MLP) with a shallow architecture [S1].

**Steps:**
1.  **Dataset Splitting:** Split the initial set of 2D training images into two subsets:
    *   Subset 1: Used to build the 2D representation (the "planes") that serves as input to the decoder.
    *   Subset 2: Used as the training set to calculate the weights of the implicit decoder [S1].
2.  **Projection (Non-Trainable):** For a 3D point $\mathbf{x}$, project it onto each of the $n$ 2D images in Subset 1 using their known camera poses. This produces a set of 2D pixel coordinates for each image.
3.  **Aggregation:** Sample the pixel values from the projected locations in all $n$ images. Aggregate these values into the vector $Z(\mathbf{x}) \in \mathbb{R}^{5k}$ [S1]. The specific aggregation function (e.g., concatenation, averaging) is not detailed, but the dimension is explicitly $5k$ [S1].
4.  **Decoding:** Pass $Z(\mathbf{x})$ through the small implicit decoder (MLP) to predict the volume density $\sigma$ and RGB color $\mathbf{c}$ [S1].
5.  **Rendering:** Use standard volumetric rendering integration along rays, using the predicted $\sigma$ and $\mathbf{c}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_plane_nerf_decode(x, images, decoder_weights):
    """
    x: 3D point (x, y, z)
    images: List of n 2D images (numpy arrays or tensors)
    decoder_weights: Parameters for the small implicit MLP
    Returns: sigma (density), color (RGB)
    """
    # Step 1: Project 3D point onto 2D images
    # Note: Camera poses for the 'images' set must be known/pre-computed.
    projected_pixels = []
    for image in images:
        # Project x onto the image plane defined by the image's camera pose.
        # This yields 2D coordinates (u, v) and potentially depth/visibility info.
        # The source does not specify the exact projection formula or handling of occlusion.
        u, v = project_3d_to_2d(x, image.camera_pose)
        
        # Sample the pixel value at (u, v) from the image.
        # Interpolation may be required if (u, v) are not integer coordinates.
        pixel_value = sample_image(image, u, v)
        projected_pixels.append(pixel_value)
    
    # Step 2: Aggregate into Z(x)
    # The source states Z(x) is in R^{5k}. The exact aggregation logic is not provided.
    # Assuming concatenation or a fixed transformation of the projected pixels.
    Z = aggregate(projected_pixels) 
    
    # Step 3: Decode
    # Pass Z through the small implicit MLP.
    # The source does not specify the MLP architecture (layers, activations).
    sigma, color = decoder_forward(Z, decoder_weights)
    
    return sigma, color
```

**Parameters:**
*   **Number of Images ($n$):** In generalization experiments, $n=50$ images per object [S1].
*   **Image Resolution:** $200 \times 200$ pixels [S1].
*   **Aggregate Vector Dimension:** $5k$, where $k$ relates to the number of images or channels [S1].

**Edge Cases and Pitfalls:**
*   **Generalization vs. Quality:** Training on a large dataset to enable generalization results in slightly worse rendering quality compared to training each object separately [S1].
*   **Occlusion:** The projection step assumes visibility. The source does not specify how occluded points are handled during projection [S1].

## Subtopics

### MultiPlaneGAN
A GAN-based generative model that uses the MultiPlaneNeRF decoder as a component. It leverages the non-trainable 2D representation to generate 3D objects [S1].

### Generalization via Image Swap
MultiPlaneNeRF can generalize to unseen objects by simply replacing the input 2D image representation without retraining the decoder [S1].

## Operating characteristics
*   **Training Time:** Significantly reduced compared to vanilla NeRF because the 3D structure is not encoded in the network weights; only a small implicit decoder is trained [S1].
*   **Inference Time:** Reduced due to the shallow decoder architecture [S1].
*   **Generalization:** Can generalize to unseen objects and classes by changing the input 2D image representation without additional training [S1].
*   **Quality Trade-off:** Achieves results comparable to state-of-the-art models (vanilla NeRF, voxel-based NSFV) but with fewer parameters. However, training on a large dataset for generalization yields slightly worse quality than per-object training [S1].
*   **Memory:** Uses fewer parameters than vanilla NeRF or voxel-based methods with larger implicit decoders [S1].

## Background
MultiPlaneNeRF was proposed to address the limitations of vanilla NeRF, specifically long training times and poor generalization to unseen data [S1]. It was introduced in the paper "MultiPlaneNeRF: Neural radiance field with non-trainable representation" [S1]. The model draws inspiration from voxel-based implicit fields and the TriPlane concept, aiming to reduce training and inference times by using non-trainable 2D image representations [S1]. The source code is available at https://github.com/gmum/MultiPlaneNeRF [S1].

## Sources
[S1] MultiPlaneNeRF: Neural radiance field with non-trainable representation, https://doi.org/10.1016/j.eswa.2025.127350, fetched 2026-10-29
