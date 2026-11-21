# Neural Style Transfer (NST)
Also called: CNN-based Style Transfer, Gatys et al. method

## Overview
Neural Style Transfer (NST) is a technique that applies the visual characteristics of an artistic style to a content image while preserving the original content structure [S1]. It differs from its parent, Adaptive Instance Normalization (AdaIN), which aligns feature map statistics for arbitrary style transfer, by relying on CNN-based feature extraction to guide the stylization process [S1].

## How it works
**Inputs:**
- Content image
- Style image

**Outputs:**
- Stylized image

**Data Structures:**
- A pre-trained convolutional neural network (CNN) is used as a fixed feature extractor [S1]. The standard implementation uses VGG-19 [S1]. Exact layouts, fields, sizes, and bit packing are not given in the sources.

**Steps:**
1. Extract feature maps from the content and style images using the fixed CNN [S1].
2. Optimize an output image to match the extracted style features while preserving the content structure [S1].
3. The iterative optimization process is computationally expensive and slow [S1].

**Formulas:**
- Not given in the sources.

**Parameters:**
- Not given in the sources.

**Pitfalls:**
- The method is computationally expensive and slow [S1].

## Subtopics

### Neural Style Transfer with Semantic Segmentation
A photorealistic style transfer method proposed by Luan et al. [S1]. It uses semantic segmentation guidance and local affine transforms to maintain scene structure while transferring artistic styles [S1].

## Operating characteristics
- **Time Complexity:** High. The iterative optimization process makes it computationally expensive and slow [S1].
- **Memory Complexity:** Not given in the sources [S1].
- **Strengths:** Achieves impressive results in artistic style transfer [S1].
- **Limitations:** Computationally expensive and slow [S1].
- **When to use:** Not given in the sources [S1].
- **When not to use:** Not given in the sources [S1].

## Background
Neural Style Transfer was pioneered by Gatys et al. [S1]. It leverages Convolutional Neural Networks (CNNs) to apply different styles to a content image [S1]. Specifically, it employs VGG-19 as a feature extractor [S1]. The method achieved impressive results in artistic style transfer but was noted for being computationally expensive and slow [S1]. Subsequent methods, such as those using feedforward networks, were developed to speed up the stylization process [S1].

## Sources
[S1] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
