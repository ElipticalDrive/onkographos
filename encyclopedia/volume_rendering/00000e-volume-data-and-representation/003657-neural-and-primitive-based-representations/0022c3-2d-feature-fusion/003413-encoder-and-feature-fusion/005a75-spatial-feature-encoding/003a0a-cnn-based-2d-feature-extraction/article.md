# CNN-based 2D Feature Extraction

Also called: CNN Feature Extraction, Per-pixel CNN Feature Extractor

## Overview
This method extracts robust, generalizable per-pixel features from input RGB images using a convolutional neural network (CNN) encoder [S1]. It solves the problem of raw RGB values being sensitive to lighting conditions and environmental noise by learning a compact feature representation that encodes geometric and photometric patterns [S1]. It differs from simple raw RGB usage by explicitly incorporating camera position awareness and using skip connections to preserve high-frequency details, distinguishing it from other spatial encoding methods that may rely on positional encoding or hash grids [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An RGB image of dimensions $H \times W \times 3$ and the camera position vector $\mathbf{c} \in \mathbb{R}^3$ [S1].
*   **Output:** A dense feature map $F \in \mathbb{R}^{H \times W \times D}$, where $D$ is the feature dimension (e.g., 128 in the primary implementation) [S1].

**Data Structures:**
The input tensor is constructed by stacking the camera position vector to each pixel. For a pixel at coordinates $(u, v)$, the input feature vector $\mathbf{x}_{u,v}$ is:
$$ \mathbf{x}_{u,v} = [\text{RGB}_{u,v}, \mathbf{c}] \in \mathbb{R}^{3+3} $$
The resulting input tensor has dimensions $H \times W \times 6$ [S1].

**Algorithm Steps:**
1.  **Input Augmentation:** For every pixel in the input RGB image, duplicate or stack the camera's 3D location vector $\mathbf{c}$ to form a 6-channel input tensor [S1].
2.  **Encoding:** Pass the augmented tensor through a CNN encoder-decoder architecture [S1].
    *   The encoder extracts hierarchical features.
    *   Skip connections are used between encoder and decoder stages to preserve high-frequency local features [S1].
    *   Fully connected (FC) layers may be integrated in the bottleneck to learn global features [S1].
3.  **Output Generation:** The network outputs the feature map $F$ of size $H \times W \times D$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cnn_feature_extraction(rgb_image: Tensor, camera_pos: Vector3) -> Tensor:
    """
    Extracts per-pixel features using a CNN with camera position awareness.
    
    Args:
        rgb_image: Tensor of shape (H, W, 3) containing RGB values.
        camera_pos: Vector3 containing the camera's xyz location.
        
    Returns:
        Tensor of shape (H, W, D) containing the extracted features.
    """
    # 1. Input Augmentation: Stack camera position to each pixel
    # camera_pos is broadcasted to match the spatial dimensions (H, W)
    # Resulting tensor shape: (H, W, 6)
    augmented_input = stack([rgb_image, broadcast(camera_pos, (H, W, 3))], axis=-1)
    
    # 2. CNN Encoder-Decoder Processing
    # The specific CNN architecture details (layer counts, kernel sizes) are not 
    # fully specified in the source, but the structure is defined as:
    # Encoder -> Bottleneck (optional FC) -> Decoder with Skip Connections
    
    # Placeholder for the specific CNN implementation defined in the source's appendix
    # The source states: "Details of the CNN module are presented in appendix"
    # and "identifying an optimal CNN module is not in the scope of this paper" [S1].
    # Therefore, the exact layer configuration is Not given in the sources.
    # We assume a standard encoder-decoder structure with skip connections [S1].
    
    features = cnn_encoder_decoder(augmented_input)
    
    return features
```

**Parameters:**
*   **Input Channels:** 6 (3 RGB + 3 Camera XYZ) [S1].
*   **Output Channels ($D$):** 128 (as shown in Figure 3 of the source) [S1].
*   **Camera Rotation/Intrinsics:** Explicitly omitted. Stacking these does not noticeably improve performance [S1].

**Edge Cases and Pitfalls:**
*   **Camera Awareness:** Do not include camera rotation or intrinsics in the input stack; empirical results show no noticeable performance improvement [S1].
*   **Lighting Sensitivity:** Raw RGB is avoided because it is sensitive to lighting and noise; the CNN learns robust patterns instead [S1].

## Subtopics

### ResUNet Image Encoder
An asymmetric ResUNet used as an image encoder $E_I$ to extract feature maps from observed images [S2]. It is chosen when a large perceptual field is required to avoid ambiguity of pixel features [S2]. It differs from the standard CNN by explicitly downsampling the input image by four times alongside feature extraction [S2]. Input is an original image $I_{4\times} \in \mathbb{R}^{4H \times 4W \times 3}$, and it outputs a feature map $F_{p'} \in \mathbb{R}^{H \times W \times n}$ and a downsampled image $I \in \mathbb{R}^{H \times W \times 3}$ [S2].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The method provides robust features that are general and representative, effectively aiding the network in practice [S1].
*   **Strengths:** Handles lighting variations and environmental noise better than raw RGB; camera position awareness allows features to be aware of relative 3D position [S1].
*   **Limitations:** The specific optimal CNN architecture is not defined in the primary source, requiring implementation choices [S1].

## Background
The method originates from the GRF (General Radiance Field) paper [S1]. It was developed to address the sub-optimality of using raw RGB values as pixel features, which are sensitive to lighting and noise [S1]. The approach leverages an encoder-decoder CNN with skip connections and camera position stacking to learn general patterns for each pixel [S1].

## Sources
[S1] GRF: Learning a General Radiance Field for 3D Representation and Rendering, https://doi.org/10.1109/iccv48922.2021.01490, fetched 2026-10-26
[S2] PI-NeRF: A Partial-Invertible Neural Radiance Fields for Pose Estimation, https://doi.org/10.1145/3581783.3612590, fetched 2026-11-08
