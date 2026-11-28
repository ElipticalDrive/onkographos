# CNN Decoder

Also called: Convolutional Texture Generator

## Overview
The CNN Decoder is a convolutional neural network that generates spatial neural texture stacks (NTS) or stylized feature maps from latent inputs, serving as the final synthesis step in neural texture-based rendering pipelines [S2, S3]. It differs from MLP-based decoders by operating on spatial grids to preserve local geometric coherence, making it suitable for generating high-resolution texture maps for UV unwrapping or style transfer [S1, unofficial][S2, S3]. It builds on [Neural Texture Decoder](6704-neural-texture-decoder.md).

## How it works
### Inputs and Outputs
*   **Human Avatar Variant:** Input is a concatenated vector of human pose $\theta$ and one-hot body part label $k_i$ [S3]. Output is a 128-dimensional spatial neural texture stack $E_i$ for body part $i$ [S3].
*   **Style Transfer Variant:** Input is a stylized feature map $F_{cs}$ derived from rendered features and style statistics [S2]. Output is a stylized novel view (RGB image) [S2].

### Data Structures
*   **Neural Texture Stack (NTS):** A 2D spatial tensor with 128 channels [S3].
*   **Pose Vector $\theta$:** Continuous vector representing SMPL pose parameters [S3].
*   **One-hot Label $k_i$:** Binary vector of size 24, where the $i$-th element is 1 [S3].

### Steps
1.  **Input Preparation (Avatar Variant):** Concatenate the pose vector $\theta$ with the one-hot body part label $k_i$ to form input vector $x_i = [\theta; k_i]$ [S3].
2.  **Batch Processing (Avatar Variant):** Concatenate inputs for all parts $i \in \{1, \dots, 24\}$ into a batch of size 24 [S3].
3.  **Forward Propagation:** Pass the input(s) through the generator network $G$ [S3]. The network consists of 5 convolutional layers for the avatar variant [S3] or 8 convolutional layers for the style transfer variant [S1, unofficial].
4.  **Local Relation Extraction (Avatar Variant):** A CNN-based module extracts local relations of the neural texture stack with respect to the human pose [S3].
5.  **Output:** Return the spatial neural texture stacks $E_i$ for UV unwrapping [S3] or the stylized image [S2].

### Parameters
*   **Number of Body Parts:** 24 [S3].
*   **NTS Channel Dimension:** 128 [S3].
*   **Network Depth:** 5 convolutional layers (avatar variant) [S3]; 8 convolutional layers (style transfer variant) [S1, unofficial].
*   **Coordinate Projection:** 1-layer 32×32×512 MLP [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_neural_textures_avatar(pose_vector, num_parts=24):
    inputs = []
    for i in range(num_parts):
        one_hot_label = [1.0 if j == i else 0.0 for j in range(num_parts)]
        x_i = pose_vector + one_hot_label  # Concatenation [G]
        inputs.append(x_i)
    batch_input = list(zip(*inputs))  # Batch of size 24 [G]
    # Forward propagate through Generator G (5 convolutional layers) [S3]
    neural_texture_stacks = apply_convolutional_generator(batch_input)  # [G]
    # Extract local relations of neural texture stack w.r.t. human pose via CNN-based module [S3]
    refined_stacks = apply_local_relation_module(neural_texture_stacks)  # [G]
    return refined_stacks

def generate_stylized_view(stylized_feature_map):
    # Forward propagate stylized feature map F_cs through CNN decoder [S2]
    stylized_image = apply_cnn_decoder(stylized_feature_map)  # [G]
    return stylized_image

def apply_convolutional_generator(x):
    # Applies 5 convolutional layers to generate 128-channel spatial tensor [S3]
    return x  # [G]

def apply_local_relation_module(x):
    # CNN-based module extracting local relations w.r.t. pose [S3]
    return x  # [G]

def apply_cnn_decoder(x):
    # CNN decoder generating final image [S2]
    return x  # [G]
```

### Edge Cases and Pitfalls
*   **Batch Size:** The avatar variant requires a batch size of exactly 24 to predict all body parts in one forward pass [S3].
*   **Input Dimensionality:** The pose vector dimension must match the expected input size of the generator network $G$ [S3].
*   **Spatial Resolution:** The output resolution of the neural texture stacks is not specified in the sources [S3].

## Subtopics

### Pose-Conditioned Convolutional Texture Generator
A specific application of the CNN Decoder for human avatar rendering [S3]. It generates 24 neural texture stacks (one per body part) conditioned on human pose and part labels [S3]. It uses a batch size of 24 to predict all textures simultaneously [S3]. It is chosen when rendering editable free-view human performance via UV unwrapping is required [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The style transfer variant allows for full-resolution feature map rendering, avoiding the computational intensity and low-resolution limitations of prior works [S1, unofficial].
*   **Strengths:** Preserves local geometric coherence via convolutional operations; efficient batch processing for multi-part textures [S3].
*   **Limitations:** Requires specific input conditioning (pose/labels); spatial resolution details are not provided [S3].

## Background
The CNN Decoder originates from two distinct lines of research [S1, unofficial][S3]. The avatar variant was introduced in "UV Volumes for Real-time Rendering of Editable Free-view Human Performance" (2023) [S3], where it is used to construct UV volumes for human point clouds. The style transfer variant was introduced in "StyleRF: Zero-Shot 3D Style Transfer of Neural Radiance Fields" (2023) [S2], where it generates stylized novel views from decomposed high-level features. The architecture overview referencing coordinate projection MLPs and 8-layer CNNs for RGB generation comes from a review of neural rendering hardware acceleration [S1, unofficial].

## Sources
[S1] Neural Rendering and Its Hardware Acceleration: A Review, https://doi.org/10.48550/arxiv.2402.00028, fetched 2026-10-26 (unofficial)
[S2] StyleRF: Zero-Shot 3D Style Transfer of Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00806, fetched 2026-10-26
[S3] UV Volumes for Real-time Rendering of Editable Free-view Human Performance, https://doi.org/10.1109/cvpr52729.2023.01595, fetched 2026-10-26
