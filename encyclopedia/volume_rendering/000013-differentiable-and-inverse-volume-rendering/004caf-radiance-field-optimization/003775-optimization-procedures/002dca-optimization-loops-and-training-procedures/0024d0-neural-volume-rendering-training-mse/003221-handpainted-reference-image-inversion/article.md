# Handpainted Reference Image Inversion

Also called: Reference Image Inversion, Semantic Colorization via Inversion

## Overview
Handpainted Reference Image Inversion is a training task where a volume rendering model learns to reproduce a user-edited 2D reference image from a 3D volume, demonstrating the ability to learn complex classification features that are difficult to define with traditional multidimensional transfer functions [S1, unofficial]. It solves the problem of tedious trial-and-error transfer function adjustments by allowing the model to automatically learn the parameters required to replicate a user-provided visual effect [S1, unofficial]. The method builds on scribble-based transfer function interactions where users modify appearance via 2D image space edits [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (e.g., Bonsai3, Pig) and a set of reference images [S1, unofficial]. The reference images are created by rendering the volume with an initial intensity-based transfer function to generate base images, then editing them in an image editor to create exemplary semantic colorization (e.g., removing spurious structures, adapting colors for consistency) [S1, unofficial].
*   **Output:** A trained rendering model (e.g., DVRNet, VNet variants, RenderNet, or Lookup table) capable of generating images that match the semantic content of the reference images [S1, unofficial].

**Data Structures:**
*   **Volume Data:** 3D scalar fields (e.g., CT data) [S1, unofficial].
*   **Reference Images:** 2D RGB images of 200 × 200 resolution [S1, unofficial].
*   **Model Architectures:** The method is evaluated on several architectures: Lookup Table (standard multidimensional transfer function representation), RenderNet (MLP-based projection layer), VNet Variants (convolutional neural networks like VNet4-4, VNetL16-4, VNetL16-17), and DVRNet (Deep Direct Volume Rendering network) [S1, unofficial].

**Numbered Steps:**
1.  **Dataset Creation:** Render the 3D volume using an initial intensity-based transfer function to generate base images [S1, unofficial]. Edit these images in an image editor to create the target reference images [S1, unofficial]. For example, in the Bonsai-H dataset, spurious green structures from a CT scanning table are removed, and colors are adapted for consistency (brown/orange for pot/stem, blue for grass) [S1, unofficial]. In the Pig-H dataset, coin visibility and brightness are adapted to bright yellow, and the coin slot is colored green [S1, unofficial].
2.  **Model Selection:** Choose a rendering model architecture (Lookup, RenderNet, VNet, or DVRNet) [S1, unofficial].
3.  **Loss Calculation:** Compute the loss between the rendered image $Y_{pred}$ and the reference image $Y$ using a combination of Mean Squared Error (MSE) and Structural Similarity Index (SSIM) [S1, unofficial]:
    $$L(Y, Y_{pred}) = L_{MSE}(Y, Y_{pred}) + (1 - SSIM(Y, Y_{pred}))$$
    The SSIM term is included to help recreate finer details [S1, unofficial].
4.  **Optimization:** Train the model to minimize the loss function for 200 epochs [S1, unofficial].
5.  **Evaluation:** Assess the model's ability to reproduce semantic attributes (e.g., distinct coloring, brightness) and consistency across views [S1, unofficial].

**Parameters:**
*   **Training Duration:** 200 epochs [S1, unofficial].
*   **Lookup Table:** Batch size = 32, Learning rate = 0.03 [S1, unofficial].
*   **RenderNet:** Batch size = 10, Learning rate = 0.05 [S1, unofficial].
*   **VNet* Variants and DVRNet:** Batch size = 2, Learning rate = 0.003 [S1, unofficial].
*   **Hardware Constraint:** Batch sizes were adapted to maximize usage of 32GB GPU memory [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Lookup Table Limitations:** Converges to a minimum achieving only general visual qualities, failing to find an adequate transfer function for complex semantic colorization [S1, unofficial].
*   **RenderNet Issues:** Suffers from excessive overblurring due to low image resolution during projection and lacks explicit opacity handling, leading to inconsistent coloring between views [S1, unofficial].
*   **VNet Limitations:** May assign general semantics but miss specific coloring details (e.g., tree stem, grass) and object brightness (e.g., coin brightness) [S1, unofficial].
*   **DVRNet Limitations:** While performing best overall, it may have minor issues with consistent coloring (e.g., green coloring for the coin slot) across all views [S1, unofficial].

## Subtopics

### [Inverse Volume Exploration](7431-inverse-volume-exploration.md)
A technique that approximates a desired rendering setting (viewpoint, color, opacity, light direction/magnitude) from a single reference image by optimizing transformation parameters of a pre-trained composed iVR-GS model [G]. It solves the problem of tedious trial-and-error transfer function adjustments by allowing the model to automatically learn the parameters required to replicate a user-provided visual effect [G]. This method enables editing of lighting and appearance via optimization, achieving perceptual fidelity even from reference images [G].

## Operating characteristics
*   **Time Complexity:** DVRNet shows considerably better training speed compared to full VNet* variants, attributed to the reduced cost of performing upsampling [S1, unofficial].
*   **Memory Complexity:** Batch sizes are constrained by available GPU memory (e.g., 32GB) [S1, unofficial]. Lookup and RenderNet can train with higher batch sizes (32 and 10 respectively) compared to VNet and DVRNet (batch size 2) [S1, unofficial].
*   **Measured Performance:** DVRNet and VNetL16-17 perform very similarly regarding LPIPS and FID metrics, while VNetL16-4 performs best in terms of SSIM on the Bonsai3-H dataset [S1, unofficial].
*   **Strengths:** Capable of learning complex classification features and semantic colorization that are difficult to define with traditional multidimensional transfer functions [S1, unofficial].
*   **Limitations:** Performance varies significantly by architecture; Lookup tables and RenderNet are unsuitable for complex semantic tasks [S1, unofficial]. Consistency across views can be challenging for some architectures [S1, unofficial].
*   **When to Use:** When the goal is to replicate complex semantic colorization or visual effects from hand-painted reference images rather than pure photometric accuracy [S1, unofficial].
*   **When Not to Use:** When simple intensity-based transfer functions are sufficient, or when strict consistency across all views is required without the computational overhead of deep learning models [G].

## Background
The method was introduced in the paper "Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images" [S1, unofficial]. It builds on previous work involving scribble-based transfer function interactions where users can modify the transfer function via interactions in the 2D image space [S1, unofficial]. The authors created two new datasets, Bonsai-H and Pig-H, based on the original Bonsai3 and Pig datasets to demonstrate the power of deeper architectures in learning complex classification features [S1, unofficial].

## Sources
[S1] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
