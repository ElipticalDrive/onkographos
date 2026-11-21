# Reference Image Guided Colorization

Also called: Reference-Based Color Transfer, Deep Image Analogy for Volume Rendering

## Overview
Reference Image Guided Colorization automates the colorization of monochromatic volume data (e.g., CT/MRI) by transferring semantic color attributes from a database of semantically similar reference images, such as histological sections [S1, unofficial]. It distinguishes itself from traditional volume rendering by replacing manual transfer function tuning with automatic deep semantic correspondence, and from other learning-based methods by requiring only a small set of reference images rather than large-scale training datasets [S1, unofficial].

## Shared principles
The family operates on the principle of transferring visual attributes (color, texture, style) from a source reference image to a target monochromatic image using deep feature matching, followed by isolation of color components for volumetric mapping.

1.  **Deep Feature Extraction:** Both target ($T$) and reference ($R$) images are processed through a pre-trained Convolutional Neural Network (CNN), typically VGG-19, to extract multi-level feature maps [S1, unofficial].
2.  **Semantic Correspondence:** A dense matching algorithm (e.g., PatchMatch) aligns features from $T$ and $R$ in the deep feature domain to construct an intermediate image $T'$ that retains the structure of $T$ but adopts the visual attributes of $R$ [S1, unofficial].
3.  **Color Isolation:** A filtering technique (e.g., Guided Image Filter) is applied to $T'$ to discard texture/style artifacts from $R$, preserving only the transferred color information as the final colorized image $C$ [S1, unofficial].
4.  **Volumetric Mapping:** The RGB channels of the volume are derived from $C$, while the alpha (opacity) channel is derived directly from the luminance of the original target data $T$ [S1, unofficial].

## Subtopics

### [Reference Based Color Transfer for Medical Volume Rendering](11552-reference-based-color-transfer-for-medical-volume-rendering.md)
This method automates the colorization of monochromatic medical volume data (CT/MRI) by transferring color attributes from a database of semantically similar colored reference images, such as histological cryosections, thereby eliminating the need for manual transfer function tuning. It distinguishes itself from traditional volume rendering by replacing user-defined color/opacity parameters with automatic deep semantic correspondence and luminance-based alpha mapping, and from other learning-based colorization methods by requiring only a small set of reference images rather than large-scale training datasets. See [Reference Based Color Transfer for Medical Volume Rendering](11552-reference-based-color-transfer-for-medical-volume-rendering.md).

### [Trained Exposure Application](13959-trained-exposure-application.md)
Trained Exposure Application is a post-processing step in volume rendering that applies a learned per-image 3x3 color transform and 3D bias vector to the rendered RGB image. It serves to match the exposure conditions of the rendered output to those present in the training data, ensuring visual consistency. It differs from its parent, Reference Image Guided Colorization, by applying a direct linear color correction rather than a complex semantic style transfer, and from its sibling, Reference Based Color Transfer, by relying on pre-computed per-viewpoint parameters rather than real-time reference image matching. See [Trained Exposure Application](13959-trained-exposure-application.md).

### Guided Image Filter Colorization
A colorization technique applied after gamma encoding to transfer color from a reference image to a target image [S1, unofficial]. Parameters: Radius $r = 16$, regularization term $\epsilon = 2$ [S1, unofficial].

### Fast Global Smoother Filter Colorization
A colorization technique applied after gamma encoding to transfer color from a reference image to a target image [S1, unofficial]. Parameters: Lambda $\lambda = 0.2$, Alpha $\alpha = 1.8$, and Lambda $\lambda = 32$ [S1, unofficial].

### Color Transfer
A simple technique applied to improve the color match between final renderings and the input style image [S2].

## Background
The core methodology originates from "Deep Image Analogy," which uses deep features for image-to-image translation [S1, unofficial]. The application to medical volume rendering was proposed to address the lack of ground-truth colorization and the difficulty of manual transfer function tuning [S1, unofficial]. The reference database often utilizes visible human project data, such as 696 cryosection images for thorax/abdomen [S1, unofficial].

## Sources
[S1] Reference Based Color Transfer for Medical Volume Rendering, https://doi.org/10.48550/arxiv.2210.08083, fetched 2026-10-26 (unofficial)
[S2] ARF: Artistic Radiance Fields, https://core.ac.uk/works/391332776, fetched 2026-10-30
