# Affine-Transformations Method
Also called: Hybrid method, Image warping method

## Overview
The Affine-Transformations Method is a hybrid volume rendering technique that uses affine transformations to warp volume slices or the entire volume into the image plane so that data samples become pixel-aligned, after which the transformed data is composited to calculate visibility [S1, unofficial]. It is distinct from pure feed-backward (ray-casting) methods, which cast rays into the data set, and pure feed-forward (element-tossing) methods, which map data elements directly to the image plane [S1, unofficial]. By aligning samples with pixels before compositing, it bridges the gap between these two extremes, leveraging the efficiency of feed-forward warping while maintaining the visibility handling of feed-backward approaches.

## Shared principles
The family shares the core mechanism of transforming volume data via affine operations to achieve pixel alignment before compositing. This process generally involves two main phases: warping and compositing.

1.  **Warping (Resampling):** The volume data (or individual slices) is resampled using affine transformations. The goal is to transform the data such that the voxel samples align with the pixel grid of the output image. This often involves a two-pass image warping approach [S1, unofficial]. To avoid resampling artifacts, particularly information loss during minification, it is recommended to perform all magnification passes before minification passes [S1, unofficial].
2.  **Compositing:** Once the data is pixel-aligned, the renderer scans the depth columns of pixels to determine the final color. This step typically involves compositing the transformed data, often using multiple data sets (such as substance, opacity, and gradient) to generate shaded slices before final composition [S1, unofficial].

The method assumes that the viewing transformation can be approximated or decomposed into affine components suitable for this warping strategy. While the parent Shear-warp method factorizes the view into shear and warp steps, the Affine-Transformations family focuses on the general application of affine warping to achieve pixel alignment, which can be applied to slices or the whole volume depending on the specific variant.

## Subtopics

### Drebin-Carpenter-Hanrahan Warping
This is the foundational variant of the affine transformation method, introduced by Drebin, Carpenter, and Hanrahan in 1988 [S1, unofficial]. It is a feed-forward algorithm that uses conventional two-pass image warping to resample volume elements in each slice so the resulting slices are pixel-aligned [S1, unofficial]. The method converts the warped data set into multiple data sets: substance, opacity, gradient, and possibly a mask [S1, unofficial]. These multiple data sets are then used to generate shaded slices, which are composited together to form the final image [S1, unofficial].

### World-Space Affine Mixture
A baseline warping variant where the affine mixture is performed in world space rather than warped space [S2]. In this model, the affine transform $A_i(\mathbf{x})$ and the normalized mixture weight $a_i(\mathbf{x})$ are defined as:

$$
A_i(\mathbf{x}) = \mathbf{R}_i(\mathbf{s}_i \circ (\mathbf{x} - \mathbf{t}_i))
$$

$$
a_i(\mathbf{x}) = \frac{w_i(\mathbf{x})}{\sum_j w_j(\mathbf{x})}
$$

where $\mathbf{R}_i$ is rotation, $\mathbf{s}_i$ is scale, $\mathbf{t}_i$ is translation, $w_i$ is the weight volume, and $a_i$ is the normalized mixture weight [S2]. The primary pitfall of this approach is that since mixture weights are not warped, the decoder must match any motion of the template by moving the mixture weights, which is less efficient than warping the weights in warp space [S2].

### [Warp-Space Affine Mixture](4115-warp-space-affine-mixture.md)
A warping method where the affine mixture weights are computed in warped space [4115-warp-space-affine-mixture.md]. This variant outperforms models with no warping, convolutional warps, or world-space mixture [4115-warp-space-affine-mixture.md]. By computing weights in the warped space, the model better accounts for the geometric transformations, leading to higher quality results in dynamic scenes such as moving hands, swinging hair, and dry ice smoke.

## Background
The Affine-Transformations Method emerged in the late 1980s as a hybrid approach to volume rendering. The seminal work was presented by Drebin, Carpenter, and Hanrahan in 1988, introducing the feed-forward image warping technique [S1, unofficial]. This work was part of the earlier Pixar research into parallel volume rendering. Hanrahan later extended the method in 1990 to a three-pass algorithm, developing the affine-mapping equations and addressing resampling artifacts by ordering magnification passes before minification passes [S1, unofficial]. The method has since been adapted in various forms, including modern neural volume rendering techniques that utilize affine mixture models for dynamic scene representation [S2].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S2] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
