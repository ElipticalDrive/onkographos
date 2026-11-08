# 3D Convolutional Neural Networks (3D CNNs)
Also called: 3D CNN

## Overview
3D CNNs extend the 2D sliding window operation to volumetric grids to extract and reconstruct spatial features for volume Super-Resolution (SR) and segmentation [S1, unofficial]. They are distinguished from their siblings by using local receptive fields that naturally match the spatial locality of volumetric features, making them the dominant architecture for these tasks [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A low-resolution 3D volume tensor (typically $H \times W \times D$).
*   Output: A high-resolution 3D volume tensor (typically $sH \times sW \times sD$, where $s$ is the scale factor).

**Data Structures:**
*   Tensors: Multi-dimensional arrays representing the volume.
*   Filters: 3D convolutional kernels with weights $w_{i,j,k}$.

**Method:**
The architecture typically employs an encoder-decoder structure [S1, unofficial].
1.  **Encoding (Downsampling):** The input volume passes through a series of 3D convolutional layers followed by downsampling operations (e.g., max pooling or strided convolutions). Each stage extracts higher-level spatial features while reducing spatial dimensions [S1, unofficial].
2.  **Decoding (Upsampling):** The encoded features pass through upsampling stages (e.g., transposed convolutions or interpolation) to reconstruct the spatial resolution [S1, unofficial].
3.  **Convolution:** 3D convolutions slide a window over the volumetric grid. For an input volume $I$ and kernel $K$, the output $O$ at position $(x,y,z)$ is computed as:
    $$ O(x,y,z) = \sum_{i,j,k} I(x+i, y+j, z+k) \cdot K(i,j,k) $$
    where $i,j,k$ are the kernel offsets [S1, unofficial].

**Parameters:**
*   Kernel size: Typically $3 \times 3 \times 3$ or $5 \times 5 \times 5$.
*   Scale factor: Determined by the number of upsampling layers.

**Edge Cases:**
*   Boundary conditions: Padding (e.g., zero-padding) is required to maintain spatial dimensions during convolution [G].
*   Memory: 3D convolutions are memory-intensive due to the cubic growth of receptive fields [G].

## Subtopics

### U-Net
The U-Net variant adds skip connections between corresponding encoder and decoder stages [S1, unofficial]. This allows the decoder to recover fine-grained spatial details lost during downsampling by concatenating feature maps from the encoder at the same resolution level [S1, unofficial]. It is chosen when preserving spatial precision is critical for SR or segmentation tasks [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K^3 \cdot C_{in} \cdot C_{out})$ per layer, where $N$ is the number of voxels, $K$ is kernel size, and $C$ are channel counts.
*   **Memory Complexity:** High, due to storing intermediate feature maps for skip connections (in U-Net) and large 3D kernels.
*   **Strengths:** Local receptive fields match volumetric spatial locality; effective for SR and segmentation [S1, unofficial].
*   **Limitations:** Can oversmooth outputs if optimized with MSE alone; computationally expensive for large volumes [S1, unofficial].
*   **When to use:** Volume SR and segmentation where spatial locality is key [S1, unofficial].
*   **When not to use:** When global context is more important than local features, or when computational resources are severely limited.

## Background
3D CNNs were developed as an extension of 2D CNNs to handle volumetric data [S1, unofficial]. They have become dominant in scientific volume analysis, particularly for Super-Resolution and segmentation tasks [S1, unofficial]. The U-Net architecture, originally for biomedical image segmentation, has been adapted for 3D volumes [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
