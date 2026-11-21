# CNN Post-Processing Network
Also called: Lightweight CNN
## Overview
A post-processing convolutional neural network applied to MLP-decoded results to reduce boundary artifacts [S1]. It sets itself apart from its siblings by focusing on spatial consistency of the final image rather than field prediction or sigma estimation, using neighboring regions to improve texture detail without causing multiview inconsistency [S2].
## How it works
**Inputs and Outputs:**
*   Input: The rendered image (or feature map) produced by the preceding MLP-based volume rendering step.
*   Output: The refined image with reduced boundary artifacts and enhanced texture detail.

**Data Structures:**
*   The CNN consists of five convolution layers, each with 32 channels [S1].
*   No skip connections are added [S1].
*   The network is shallow, typically 3 layers in some implementations [S2].

**Steps:**
1.  **Input Padding:** Apply "reflectance" padding to the input features to handle boundary conditions for convolution [S2].
2.  **Convolutional Processing:** Pass the padded input through the convolutional layers. Each layer uses 3x3 kernels and ReLU activations [S2].
3.  **Output Trimming:** Remove the padding values from the outputs of the convolutional layers to prevent checkerboard artifacts [S2].
4.  **Feature-to-RGB Conversion:** Apply a feed-forward perceptron layer to convert the representation from feature space to RGB space [S2].

**Formulas:**
*   Loss Function (for training): Mean Squared Error (MSE) between the MLP-decoded results and the ground-truth (GT) volume [S1].
    $$ L = \text{MSE}(I_{MLP}, I_{GT}) $$
    Where $I_{MLP}$ is the image decoded by the MLP and $I_{GT}$ is the ground truth.

**Parameters:**
*   CNN Architecture: 5 convolution layers, 32 channels each [S1].
*   Kernel Size: 3x3 [S2].
*   Activation: ReLU [S2].
*   Quantization: 9 bits applied at the end of CNN training without subsequent fine-tuning [S1].

**Edge Cases and Pitfalls:**
*   **Checkerboard Artifacts:** Prevented by removing padding values in the outputs [S2].
*   **Multiview Inconsistency:** Avoided because the CNN is very shallow and performs no upsampling [S2].
*   **Artifact Removal Limit:** Limited capacity does not guarantee removal of boundary artifacts when distortion is significant; remaining artifacts may still negatively impact rendering quality [S1].
*   **Training Overhead:** Adds approximately 5 minutes to encoding time and 100 KB to model size [S1].
*   **Decoding Overhead:** Adds approximately 50 seconds to decoding time due to sequential inference [S1].

## Subtopics

### Boundary Artifact Mitigation
A specific application of the CNN to mitigate rendering artifacts caused by inconsistent outputs between adjacent disjoint volumetric blocks along their boundaries. It sets itself apart by targeting the multiscale spatiotemporal representation's block-boundary issues. Choose this when using ECNR or similar block-based neural representations. It applies a lightweight CNN module to the output of the finest scale MLPs. The CNN consists of five convolution layers, each with 32 channels, with no skip connections. It is trained using an MSE loss between MLP-decoded results and ground-truth volume. After training, quantization with 9 bits is applied without fine-tuning. [Boundary Artifact Mitigation](7251-supplementary-feature-input-for-super-resolution.md)

### [Supplementary Feature Input for Super-Resolution](7251-supplementary-feature-input-for-super-resolution.md)
A technique where the super-resolution network accepts multiple feature maps alongside the low-resolution color image to improve reconstruction quality, specifically adapted for volumetric rendering. It sets itself apart from standard image super-resolution by incorporating volumetric-specific auxiliary data (depth, color-opacity vectors) to resolve ambiguities inherent in volume rendering, such as undefined depth in transparent regions. [Supplementary Feature Input for Super-Resolution](7251-supplementary-feature-input-for-super-resolution.md)

### [2D Refinement Layers](12626-2d-refinement-layers.md)
2D Refinement Layers is an optional post-processing module applied to rendered images to enhance texture detail and prevent "texture sticking" artifacts under viewpoint changes. It builds on [CNN Post-Processing Network](7179-cnn-post-processing-network.md) by specifically employing alias-free convolutional layers to ensure temporal and spatial consistency when the camera moves. It addresses the limitation of standard CNNs where high-frequency details may appear to stick to the image plane rather than the 3D geometry. [2D Refinement Layers](12626-2d-refinement-layers.md)

## Operating characteristics
*   **Time Complexity:** Decoding adds approximately 50 seconds due to sequential inference [S1]. Despite this, it remains significantly faster than neurcomp [S1].
*   **Memory Complexity:** Adds approximately 100 KB to model size [S1].
*   **Strengths:** Reduces boundary artifacts; improves texture detail; avoids multiview inconsistency; prevents checkerboard artifacts.
*   **Limitations:** Limited capacity may not remove significant distortion artifacts; adds sequential inference time; does not process each timestep independently in multiscale representations [S1].
*   **When to Use:** When using block-based neural representations (like ECNR) where boundary artifacts are prominent, and multiview consistency is critical.
*   **When Not to Use:** When extreme distortion exists that exceeds the CNN's capacity to correct, or when minimal decoding overhead is required.

## Background
Originated in the context of Efficient Compressive Neural Representation (ECNR) for time-varying volumetric datasets [S1]. It was introduced to address the issue of inconsistent outputs between adjacent blocks in multiscale spatiotemporal representations [S1]. The method was also explored in AligNeRF for high-fidelity neural radiance fields [S2].
## Sources
[S1] ECNR: Efficient Compressive Neural Representation of Time-Varying Volumetric Datasets, https://doi.org/10.1109/pacificvis60374.2024.00017, fetched 2026-10-26
[S2] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
