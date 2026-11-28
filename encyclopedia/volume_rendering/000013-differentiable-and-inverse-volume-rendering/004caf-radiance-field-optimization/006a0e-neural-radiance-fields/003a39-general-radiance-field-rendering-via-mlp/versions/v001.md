# General Radiance Field Rendering via MLP

Also called: MLP Radiance Field with Aggregated Features

## Overview
This method renders novel views by training a Multi-Layer Perceptron (MLP) to predict volumetric density and RGB color for 3D points, forming a "General Radiance Field" (GRF). It differs from standard Neural Radiance Fields (NeRF) by using aggregated 3D point features as the primary spatial input to the MLP, rather than raw 3D coordinates, while retaining the viewing direction as a secondary input. It solves the problem of generalizable 3D representation and rendering from 2D images without requiring explicit 3D ground truth data.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 2D images with known camera poses. For rendering, the inputs are a 3D point $p$ and a viewing direction $V_p$.
*   **Outputs:** Volumetric density $\sigma(t)$ and RGB color $c(t)$ for points along a ray, resulting in a rendered 2D image pixel value.

**Data Structures:**
*   **Aggregated Point Features ($\bar{F}_p$):** A feature vector representing the 3D point $p$. The exact dimension and generation method are not specified in the source, but it replaces the raw coordinate vector $[x, y, z]$ used in NeRF.
*   **Viewing Direction ($V_p$):** A 3D vector $[x_v, y_v, z_v]$ representing the direction from the camera center to point $p$.
*   **MLP:** A neural network architecture strictly following the design in NeRF [29]. It takes the concatenated input $[\bar{F}_p, V_p]$ and outputs density and color.

**Numbered Steps:**

1.  **Positional Encoding:** Apply the positional encoding function $\gamma(\cdot)$ proposed in NeRF [29] to both the aggregated point features $\bar{F}_p$ and the viewing direction $V_p$.
    $$ \text{input} = [\gamma(\bar{F}_p), \gamma(V_p)] $$
2.  **MLP Forward Pass:** Feed the encoded input into the MLP to predict the volumetric density $\sigma$ and RGB color $c$ for the query point.
    $$ [\sigma, c] = \text{MLP}(\text{input}) $$
3.  **Ray Casting:** For each pixel in the target image, cast a ray from the camera center through the pixel into the 3D space.
4.  **Hierarchical Sampling:** Estimate the volume rendering integral using numerical quadrature with a hierarchical sampling strategy, strictly following the method in NeRF [29]. This involves sampling points along the ray, potentially using coarse and fine networks (though the source states the MLP design follows NeRF, it does not explicitly detail the two-network hierarchy, only that the *strategy* is followed).
5.  **Volume Rendering:** Compute the final pixel color $rgb$ by integrating the density and color along the ray:
    $$ rgb = \int_{t_n}^{t_f} T(t) c(t) \sigma(t) dt $$
    where the transmittance $T(t)$ is:
    $$ T(t) = \exp\left(-\int_{t_n}^{t} \sigma(s) ds\right) $$
    Here, $t_n$ and $t_f$ are the near and far clipping planes, $c(t)$ is the color at point $t$, and $\sigma(t)$ is the volume density at point $t$.
6.  **Optimization:** Minimize the L2 loss between the rendered RGB values and the ground truth RGB values from the input images. The entire network is trained end-to-end.

**Parameters:**
*   **Loss Function:** L2 loss between rendered and ground truth RGB.
*   **Positional Encoding:** As defined in NeRF [29].
*   **Sampling:** Hierarchical sampling strategy as defined in NeRF [29].
*   **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** The ray integration is bounded by near and far clipping planes ($t_n, t_f$).
*   **Degenerate Input:** Not given in the sources.

## Subtopics

### [Visibility and Outgoing Radiance MLP](16668-visibility-and-outgoing-radiance-mlp.md)
A neural network representation that replaces costly recursive path tracing for indirect illumination by learning a continuous function for visibility and outgoing radiance.

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On ShapeNetv2 chair and car classes, the method (GRF) achieved a PSNR of 22.65. For comparison, SRNs achieved 24.48. Other metrics (SSIM, LPIPS, etc.) are provided in the source but not detailed here beyond the PSNR comparison.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Can synthesize novel 2D images directly by querying points along light rays, trainable only with 2D images without requiring 3D data.
*   **Limitations:** The source does not specify limitations compared to NeRF other than the architectural difference in input.
*   **When to use:** When a general radiance field representation is desired that leverages aggregated features rather than raw coordinates.
*   **When not:** When raw coordinate-based spatial encoding is preferred or when higher PSNR is required (SRNs performed better in the cited experiment).

## Background
The method was introduced in the paper "GRF: Learning a General Radiance Field for 3D Representation and Rendering" [S1]. It builds on the principles of Radiance-field optimization [44-radiance-field-optimization.md] and specifically adapts the MLP architecture from NeRF [4972-neural-radiance-fields.md]. The key innovation is the substitution of raw 3D coordinates with aggregated point features as the input to the radiance field MLP.

## Sources
[S1] GRF: Learning a General Radiance Field for 3D Representation and Rendering, https://doi.org/10.1109/iccv48922.2021.01490, fetched 2026-10-26
