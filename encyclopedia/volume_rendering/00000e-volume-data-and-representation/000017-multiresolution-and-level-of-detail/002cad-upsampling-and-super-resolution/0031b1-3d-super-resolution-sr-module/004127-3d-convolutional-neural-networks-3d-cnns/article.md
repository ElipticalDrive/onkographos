# 3D Convolutional Neural Networks (3D CNNs)
Also called: 3D CNN

## Overview
3D CNNs extend the 2D sliding window operation to volumetric grids, using encoder-decoder architectures to stack downsampling and upsampling stages for extracting and reconstructing spatial features [S1, unofficial]. They are the dominant architectures for volume Super-Resolution (SR) and segmentation because their local receptive fields naturally match the spatial locality of volumetric features [S1, unofficial]. Unlike sibling methods like [SSR-VFD](10422-ssr-vfd.md) or [Hierarchical Multi-scale SR](10444-hierarchical-multi-scale-sr.md), plain 3D CNNs rely on standard convolutional stacks without specialized implicit neural representations or hierarchical multi-scale query mechanisms, focusing instead on direct voxel-grid feature manipulation.

## How it works
**Inputs and Outputs:**
- Input: A low-resolution 3D volume tensor $V_{in} \in \mathbb{R}^{D \times H \times W \times C_{in}}$, where $D, H, W$ are spatial dimensions and $C_{in}$ is the number of input channels.
- Output: A high-resolution 3D volume tensor $V_{out} \in \mathbb{R}^{D' \times H' \times W' \times C_{out}}$, where $D', H', W'$ are the upsampled spatial dimensions.

**Data Structures:**
- **Feature Map:** A 4D tensor representing activations. Layout is typically $(Batch, Channel, Depth, Height, Width)$.
- **Convolution Kernel:** A 5D tensor $K \in \mathbb{R}^{C_{out} \times C_{in} \times k_d \times k_h \times k_w}$, where $k_d, k_h, k_w$ are kernel sizes in depth, height, and width.
- **Bias:** A 1D tensor $b \in \mathbb{R}^{C_{out}}$.

**Numbered Steps:**
1. **Input Padding:** Pad the input volume $V_{in}$ with zeros to maintain spatial dimensions if 'SAME' padding is used, or allow reduction if 'VALID' padding is used. The padding size $p$ for a kernel of size $k$ and stride $s$ is calculated as $p = \lfloor \frac{(n-1)s + k - n}{2} \rfloor$ for 'SAME' padding, where $n$ is the input dimension [G].
2. **Convolution Operation:** For each output channel $c_{out}$, compute the dot product between the kernel $K_{c_{out}}$ and the corresponding input region. The value at output position $(d, h, w)$ for channel $c_{out}$ is:
   $$
   V_{out}[d, h, w, c_{out}] = \sum_{c_{in}=0}^{C_{in}-1} \sum_{k_d=0}^{k_d-1} \sum_{k_h=0}^{k_h-1} \sum_{k_w=0}^{k_w-1} V_{in}[d+k_d, h+k_h, w+k_w, c_{in}] \cdot K[c_{out}, c_{in}, k_d, k_h, k_w] + b[c_{out}]
   $$
3. **Activation:** Apply a non-linear activation function (e.g., ReLU) to the result of the convolution: $V_{act} = \max(0, V_{out})$.
4. **Downsampling (Encoder):** If an encoder-decoder architecture is used, apply strided convolutions or max-pooling to reduce spatial dimensions. Strided convolution reduces dimensions by a factor equal to the stride $s$: $n_{out} = \lfloor \frac{n_{in} - k + 2p}{s} \rfloor + 1$.
5. **Upsampling (Decoder):** If an encoder-decoder architecture is used, upsample the feature maps. This can be done via transposed convolutions (deconvolution) or interpolation followed by convolution. Transposed convolution increases spatial dimensions by a factor related to the stride.
6. **Skip Connections (Optional):** If using a U-Net variant, concatenate encoder feature maps with corresponding decoder feature maps at the same resolution level to preserve spatial details [S1, unofficial].
7. **Final Reconstruction:** Apply a final convolutional layer to map the last feature map to the desired output channels $C_{out}$.

**Parameters with Typical Values:**
- Kernel size ($k_d, k_h, k_w$): Typically $3 \times 3 \times 3$ or $5 \times 5 \times 5$.
- Stride ($s$): Typically 1 for feature extraction, 2 for downsampling/upsampling.
- Number of feature maps: Varies by layer, often doubling at each downsampling stage (e.g., 32, 64, 128, 256).
- Activation: ReLU is common.

**Edge Cases and Pitfalls:**
- **Boundary Effects:** Convolution near volume boundaries may produce artifacts if padding is not handled correctly. Zero-padding is standard but can introduce dark borders.
- **Memory Usage:** 3D convolutions are memory-intensive. The memory footprint scales with $D \times H \times W \times C_{in} \times C_{out} \times k_d \times k_h \times k_w$. Large volumes may require tiling or reduced batch sizes.
- **Degenerate Input:** Empty or single-voxel volumes may cause errors in pooling or convolution operations if not handled.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def conv3d(input_volume, kernel, bias, stride=1, padding=0):
    """
    Performs a 3D convolution.
    
    Args:
    input_volume: np.ndarray of shape (D, H, W, C_in)
    kernel: np.ndarray of shape (C_out, C_in, k_d, k_h, k_w)
    bias: np.ndarray of shape (C_out,)
    stride: int, stride for convolution
    padding: int, padding size
    
    Returns:
    output_volume: np.ndarray of shape (D_out, H_out, W_out, C_out)
    """
    D, H, W, C_in = input_volume.shape
    C_out, _, k_d, k_h, k_w = kernel.shape
    
    # Calculate output dimensions
    D_out = (D + 2 * padding - k_d) // stride + 1
    H_out = (H + 2 * padding - k_h) // stride + 1
    W_out = (W + 2 * padding - k_w) // stride + 1
    
    # Initialize output volume
    output_volume = np.zeros((D_out, H_out, W_out, C_out))
    
    # Pad input volume
    padded_input = np.pad(input_volume, ((padding, padding), (padding, padding), (padding, padding), (0, 0)), mode='constant')
    
    for c_out in range(C_out):
        for d in range(D_out):
            for h in range(H_out):
                for w in range(W_out):
                    # Extract region
                    d_start = d * stride
                    h_start = h * stride
                    w_start = w * stride
                    region = padded_input[d_start:d_start+k_d, h_start:h_start+k_h, w_start:w_start+k_w, :]
                    
                    # Compute dot product
                    dot_product = np.sum(region * kernel[c_out, :, :, :, :])
                    output_volume[d, h, w, c_out] = dot_product + bias[c_out]
                    
    return output_volume

def relu(x):
    return np.maximum(0, x)

def encoder_decoder_3d_cnn(input_volume, encoder_kernels, encoder_biases, decoder_kernels, decoder_biases):
    """
    Simple encoder-decoder 3D CNN.
    
    Args:
    input_volume: np.ndarray of shape (D, H, W, C_in)
    encoder_kernels: list of kernel arrays for encoder layers
    encoder_biases: list of bias arrays for encoder layers
    decoder_kernels: list of kernel arrays for decoder layers
    decoder_biases: list of bias arrays for decoder layers
    
    Returns:
    output_volume: np.ndarray of shape (D_out, H_out, W_out, C_out)
    """
    current_volume = input_volume
    
    # Encoder
    for i in range(len(encoder_kernels)):
        current_volume = conv3d(current_volume, encoder_kernels[i], encoder_biases[i], stride=2, padding=1)
        current_volume = relu(current_volume)
        
    # Decoder
    for i in range(len(decoder_kernels)):
        # Upsample (simple nearest neighbor for demonstration)
        current_volume = np.repeat(np.repeat(np.repeat(current_volume, 2, axis=0), 2, axis=1), 2, axis=2)
        current_volume = conv3d(current_volume, decoder_kernels[i], decoder_biases[i], stride=1, padding=1)
        current_volume = relu(current_volume)
        
    return current_volume
```

## Subtopics

### [U-Net](10523-u-net.md)
The U-Net variant is an encoder-decoder convolutional neural network architecture designed for volumetric segmentation and super-resolution. It extends standard 3D CNNs by adding skip connections between corresponding encoder and decoder stages, allowing the network to combine high-level semantic context with low-level spatial details. This structure distinguishes it from plain 3D CNNs by mitigating the loss of fine-grained spatial information during downsampling, making it particularly effective for tasks requiring precise boundary localization in 3D volumes.

### [Voxel Super-Resolution Decoder](11760-voxel-super-resolution-decoder.md)
A decoder architecture designed to upsample low-resolution voxel grids to higher resolutions for super-resolution tasks, serving as a fast alternative to querying implicit MLPs for higher resolution data. It distinguishes itself from sibling methods like [U-Net](10523-u-net.md) by utilizing a specific sequence of 3D convolutional layers with instance normalization and ReLU activations, rather than a symmetric encoder-decoder skip-connection structure.

### SRDenseNet
SRDenseNet is a deep network used as a super-resolution module to reconstruct high-resolution (HR) images from low-resolution (LR) inputs, selected for its generalization ability on medical images [S2, unofficial]. The SRDenseNet-All variant, used in implementation, consists of four stages: low-level features, high-level features, deconvolution layers, and a reconstruction layer [S2, unofficial]. The low-level feature stage receives an LR image. There are eight dense blocks learning high-level features. Each dense block has eight convolution layers, each producing 16 feature maps (128 feature maps per block) [S2, unofficial]. The bottleneck layer decreases feature maps to a compact model. A deconvolution layer upscales resolution from 128x128 to 512x512. A final convolutional layer with a 3x3 kernel reduces output channels to a single channel [S2, unofficial]. Input resolution for upscaling is 128x128, and output resolution is 512x512 [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(D \cdot H \cdot W \cdot C_{in} \cdot C_{out} \cdot k_d \cdot k_h \cdot k_w)$ per convolutional layer. Encoder-decoder architectures double this due to upsampling and skip connections.
- **Memory Complexity:** $O(D \cdot H \cdot W \cdot C_{max})$ for feature maps, where $C_{max}$ is the maximum number of channels in the network. Memory usage is significantly higher than 2D CNNs due to the third dimension.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with volume size and number of channels. Parallelization on GPUs is effective but limited by memory bandwidth.
- **Strengths:** Local receptive fields match volumetric spatial locality; effective for SR and segmentation; well-established architectures.
- **Limitations:** High memory consumption; fixed receptive field size; may oversmooth details if not carefully trained (e.g., using GANs for perceptual quality [S1, unofficial]).
- **When to Use:** When volumetric spatial locality is important, and memory resources are sufficient. Suitable for SR and segmentation tasks.
- **When Not to Use:** When global context is more important than local features, or when memory is severely constrained.

## Background
3D CNNs extend the 2D sliding window operation to volumetric grids [S1, unofficial]. They are framed by the volumetric tasks they address rather than as generic definitions [S1, unofficial]. For comprehensive treatment of these architectures, see Goodfellow et al. [40, cited in S1, unofficial]. 3D CNNs and U-Net are the dominant architectures for volume SR and segmentation [S1, unofficial].

## Sources
[S1] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S2] Bournemouth University Research Online [BURO] - A Novel Implicit Neural Representation for Volume Data., https://eprints.bournemouth.ac.uk/38355/, fetched 2026-10-29 (unofficial)
