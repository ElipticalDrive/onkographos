# DEHNet

Also called: Neural Diffraction-Engineered Holography Network

## Overview
DEHNet is a convolutional neural network designed to synthesize Diffraction-Engineered Hologram (DEH) wave fields directly from an all-in-focus color image and an all-in-focus depth map [S1]. It solves the computational bottleneck of traditional DEH generation, which requires multiple varifocal images and iterative optimization, by replacing the optimization procedure with a direct inference step [S1]. Unlike its sibling method nCGH, which also uses RGB-D inputs but lacks depth-aware defocus control, DEHNet produces wave fields that correctly reconstruct defocus blur and sharp focus while suppressing interference artifacts at object boundaries [S1].

## How it works
**Inputs and Outputs**
*   **Input:** An all-in-focus color image ($I_{color}$) and an all-in-focus depth map ($D$) [S1].
*   **Output:** A DEH wave field (complex amplitude) [S1].

**Data Structures**
The network consists of 34 convolutional layers [S1].
*   **Hidden Layers:** Layers 1 through 33 are standard convolutional layers. Each layer has 12 channels [S1].
*   **Output Layer:** Layer 34 is the final layer. It includes a concatenated shortcut (skip connection) from earlier in the network [S1]. It outputs the DEH wave field [S1].
*   **Channels:** The number of channels is set to 12. This value is selected as small as possible to maximize the number of convolutions and activations under restricted computational resources, prioritizing non-linearity and wide receptive field over hidden feature capacity [S1].

**Parameters**
*   **Quantization:** After training, weights are quantized to 8-bit integers to reduce computational load during inference [S1].
*   **Training Data:** The dataset consists of 3000 different scenes [S1]. Each scene contains:
    *   21 varifocal images
    *   1 all-in-focus color image
    *   21 varifocal depth maps
    *   1 all-in-focus depth map [S1]

**Algorithm Steps**
1.  **Preprocessing:** Obtain an all-in-focus color image and an all-in-focus depth map for the target scene [S1].
2.  **Inference:** Pass the color image and depth map into the quantized DEHNet [S1].
3.  **Synthesis:** The network computes the optimal wave field directly [S1].
4.  **Encoding:** Encode the resulting amplitude-only hologram for display [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dehnet_inference(color_image, depth_map, quantized_weights):
    """
    Synthesizes a DEH wave field using the quantized DEHNet.
    
    Args:
        color_image (Tensor): All-in-focus color image.
        depth_map (Tensor): All-in-focus depth map.
        quantized_weights (List[Tensor]): Quantized 8-bit integer weights for 34 layers.
        
    Returns:
        Tensor: The synthesized DEH wave field.
    """
    # Input concatenation or processing is implied by the architecture
    # The network takes RGB-D as input
    x = concatenate(color_image, depth_map)
    
    # Layers 1 to 33: Convolutional layers with 12 channels
    for i in range(33):
        # Conv2d, 3x3 kernel, 12 channels
        x = conv2d(x, quantized_weights[i], kernel_size=3, channels=12)
        # PReLU activation is used in the residual blocks described in Fig 2
        x = prelu(x)
        
    # Layer 34: Output layer with concatenated shortcut
    # The last layer includes a concatenated shortcut (skip connection)
    # Assuming the shortcut connects to an earlier feature map, 
    # typically the input or a specific intermediate layer. 
    # Based on "concatenated shortcut", we concatenate the input or a previous feature map.
    # Note: The exact skip connection source (e.g., input vs layer 1) is not explicitly 
    # detailed in the text beyond "includes a concatenated shortcut". 
    # Standard residual/skip logic implies concatenating with a previous representation.
    # For implementation, we assume a standard skip from input or early layer.
    # Given the text "last layer which includes a concatenated shortcut", 
    # we concatenate the input feature map to the current feature map before the final conv.
    x_shortcut = concatenate(color_image, depth_map) # Placeholder for specific skip source
    x = concatenate(x, x_shortcut)
    
    # Final convolution to output DEH wave field
    # Output channels correspond to the complex wave field components
    deh_wave_field = conv2d(x, quantized_weights[33], kernel_size=3)
    
    return deh_wave_field
```

**Edge Cases and Pitfalls**
*   **Depth Map Errors:** Real-world captured depth maps often contain incorrect values or depth holes [S1]. These errors induce severe noise and distort objects in the reconstructed hologram due to interference-induced black lines at boundaries [S1]. DEHNet is designed to suppress this noise at boundaries, unlike nCGH which produces visible black lines at noisy depth interfaces [S1].
*   **Occlusion:** The method must handle occluded surfaces correctly to provide appropriate blur and sharp focus [S1].

## Operating characteristics
**Time and Memory Complexity**
*   **Inference Time:** DEHNet (quantized, qNET) achieves a frame rate of 62 Hz [S1].
*   **Total Frame Rate:** Including the encoding of amplitude-only holograms (0.89 ms), the total frame rate is 57 Hz [S1].
*   **Comparison to Optimization:** DEHNet is 8600 times faster than the optimization-based DEH method (OPT), which requires more than 1 minute per hologram [S1].
*   **Comparison to nCGH:** DEHNet is 70 times faster than nCGH [S1].

**Measured Performance**
*   **512x512 Dataset:**
    *   DEHNet (qNET): PSNR 28.5 dB, SSIM 0.894 [S1].
    *   nCGH: PSNR 22.5 dB, SSIM 0.756 [S1].
    *   DEHNet provides a 6.5 dB enhancement in PSNR and 0.15 enhancement in SSIM compared to nCGH [S1].
*   **FHD Dataset:**
    *   DEHNet (qNET): PSNR 32.7 dB, SSIM 0.939 [S1].
    *   nCGH: PSNR 27.1 dB, SSIM 0.889 [S1].
    *   DEHNet provides a 6.3 dB enhancement in PSNR and 0.07 enhancement in SSIM compared to nCGH [S1].
*   **Comparison to OPT:** DEHNet (qNET) has a PSNR of 28.5 dB, compared to OPT's 29.0 dB, representing a loss of only ~0.5 dB [S1].

**Strengths and Limitations**
*   **Strengths:** Real-time performance (62 Hz); superior image quality compared to nCGH; robustness to noisy depth maps from real-world sensors; correct depth perception and defocus blur [S1].
*   **Limitations:** Requires training data (3000 scenes); performance depends on the quality of the input depth map, though it is more robust than nCGH [S1].

## Background
DEHNet was introduced to address the impracticality of optimization-based DEH generation for real-time applications, which requires numerous varifocal images and extensive computation [S1]. It builds on the Diffraction-Engineered Holography (DEH) method, which synthesizes optimal wave fields for clear defocus blur and correct depth perception [S1]. The method was developed by the authors of the source paper, published in *Nature Communications* in 2022 [S1].

## Sources
[S1] Diffraction-engineered holography: Beyond the depth representation limit of holographic displays, https://www.nature.com/articles/s41467-022-33728-5, fetched 2026-10-29
