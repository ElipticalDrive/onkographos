# Feature Fields

Also called: Generalized NeRF, Semantic Fields

## Overview
Feature Fields generalize the standard Neural Radiance Field (NeRF) formulation by extending the output space to predict arbitrary per-point feature signals $\mathbf{f}$ in addition to volume density $\sigma$ and view-dependent color $\mathbf{c}$ [S1, S2, unofficial][S3, S4]. This extension allows the radiance field to encode semantic labels, physical properties, or latent descriptors from foundation models, enabling tasks such as zero-shot segmentation, object manipulation, and 3D detection [S1, S2, unofficial][S3, S4]. Unlike sibling methods that focus on specific applications like pose estimation [PI-NeRF](12124-pi-nerf.md) or gaze redirection [GazeNeRF](12127-gazenerf.md), this family provides the general architectural mechanism for embedding high-dimensional feature spaces into the volumetric representation [S1, S2, unofficial][S3, S4].

## Shared principles
The baseline NeRF architecture [Parent](3065-nerf-hyperparameters.md) defines a continuous volumetric scene representation $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\mathbf{x} \in \mathbb{R}^3$ is the spatial position, $\mathbf{d} \in \mathbb{R}^3$ is the viewing direction, $\sigma$ is the volume density, and $\mathbf{c}$ is the view-dependent color [G]. Feature Fields extend this by modifying the output space to predict arbitrary per-point feature signals $\mathbf{f}$ alongside density and color [S1, unofficial].

The volume rendering equation for a ray $\mathbf{r}$ with $K$ sample points is generalized to integrate these features:
$$
\hat{F}(\mathbf{r}) = \sum_{k=1}^{K} \hat{T}_k \alpha(\sigma(\mathbf{x}_k)\delta_k) \mathbf{f}(\mathbf{x}_k)
$$
where $\hat{T}_k = \exp(-\sum_{j=1}^{k-1} \sigma(\mathbf{x}_j)\delta_j)$ is the accumulated transmittance, $\alpha(\cdot)$ is the opacity function, and $\mathbf{f}(\mathbf{x}_k)$ is the feature vector at sample point $\mathbf{x}_k$ [S1, unofficial].

Supervision is achieved by reprojecting the predicted 3D features onto the 2D image plane and comparing them to corresponding 2D feature maps, mirroring the color reconstruction process [S1, unofficial]. This allows encoding semantic labels, physical properties, or latent descriptors from foundation models [S1, unofficial].

## Subtopics

### [Multi-Space Neural Radiance Field (MS-NeRF)](11707-multi-space-neural-radiance-field-ms-nerf.md)
MS-NeRF addresses the failure of standard single-space NeRFs to render reflective and refractive objects, which violate multi-view consistency in Euclidean space. It represents the scene as a group of parallel feature fields (sub-spaces) rather than a single radiance field, allowing virtual images to be placed in sub-spaces visible only from specific viewpoints. This approach serves as a general enhancement module for existing NeRF backbones with minimal computational overhead.

### [NRFF](12072-nrff.md)
NRFF encodes view-dependent specular effects by replacing positional encoding and explicit illumination models with a latent-space representation based on Anisotropic Spherical Gaussians (ASG). It predicts ASG features via a spatial MLP and compresses them into a bottleneck vector processed by a Rendering Equation Encoding (REE) function before entering the directional MLP.

### [Semantic and Feature Branches](12095-semantic-and-feature-branches.md)
This method extends the standard NeRF formulation by predicting arbitrary per-point feature signals $\mathbf{f}$ in addition to density $\sigma$ and color $\mathbf{c}$. It allows the radiance field to encode semantic labels, physical properties, or latent descriptors from foundation models, enabling tasks such as zero-shot segmentation, object manipulation, and 3D detection. Unlike sibling methods that focus on specific applications like pose estimation [PI-NeRF](12124-pi-nerf.md) or gaze redirection [GazeNeRF](12127-gazenerf.md), this family provides the general architectural mechanism for embedding high-dimensional feature spaces into the volumetric representation.

### [NeRF for Object Segmentation](12404-nerf-for-object-segmentation.md)
This method extends the standard NeRF formulation to perform object segmentation by simultaneously estimating scene geometry and predicting semantic occlusion masks. It introduces "area encoding" to represent the probability that a sampling point belongs to an occluded area, leveraging the correlation between geometry estimation and semantic label prediction. Unlike sibling methods that predict semantic labels directly as a feature vector [Siblings], this approach treats segmentation as a volumetric integration problem, ensuring view-consistent results by jointly optimizing color, depth, and area encoding losses.

### [Semantic-NeRF](14889-semantic-nerf.md)
Semantic-NeRF synthesizes semantic labels for novel views by extending the standard NeRF formulation with an additional direction-independent MLP branch. It solves the problem of 3D semantic understanding and label propagation by ensuring multi-view consistency between rendered semantic labels and ground-truth 2D annotations. It sets itself apart from siblings like [Multi-Space Neural Radiance Field (MS-NeRF)](11707-multi-space-neural-radiance-field-ms-nerf.md) and [Semantic and Feature Branches](12095-semantic-and-feature-branches.md) by being the first work to integrate semantic information directly into the NeRF architecture using a specific branch that consumes position and density features, rather than relying on separate scene/object branches or multi-space embeddings.

### [Augmented Features (Avg, Var, Color)](15000-augmented-features-avg-var-color.md)
This method augments the volumetric representation in NeRF-based 3D detection by explicitly encoding mean (average), variance, and color features into the cost volume or feature field. It solves the problem of insufficient geometric priors in standard NeRF formulations for 3D object detection by leveraging statistical moments of the radiance field and low-level color information as explicit geometry cues. It sets itself apart from sibling methods like [Semantic-NeRF](14889-semantic-nerf.md) by focusing on statistical variance and color intensity rather than semantic class labels or latent descriptors, providing a lightweight geometric prior for detection heads.

### Object Field Representation
A function $f(\mathbf{p})$ that maps a 3D point $\mathbf{p}$ to an object code vector $\mathbf{o}$, representing object ownership independent of viewing angles [S2, unofficial]. The object code $\mathbf{o}$ is defined as $H+1$ dimensional, where $H$ is a predefined maximum number of solid objects the network predicts, and the last dimension represents empty space to handle occlusion and collision [S2, unofficial]. The function $f$ is parameterized by a series of MLPs [S2, unofficial].

### N3F
A method that learns additional feature fields to realize self-supervised 3D segmentation [S3].

### NeRF Branch Feature Sampling and Augmentation
Variance of color features describes occupancy: low variance implies occupied 3D location (Lambertian assumption), high variance implies free space (different appearances from different views) [S4]. Sample points along the ray originated from the camera: $\mathbf{r}(t) = \mathbf{o} + t \times \mathbf{d}$, where $\mathbf{o}$ is the ray origin and $\mathbf{d}$ is the view direction [S4]. Augment sampled features with variance from multiple views to describe occupancy. Compute variance: $V_{var}(\mathbf{p}) = \sum_{i=1}^{M_p} (V_i(\mathbf{p}) - \bar{V}(\mathbf{p}))^2 / M_p$ [S4]. Compute mean and variance for pixel RGB values in the same way as for deep features [S4]. Pass sampled augmented features into NeRF MLPs to generate density and color [S4]. Density and latent features are computed as: $\sigma(\mathbf{p}), \hat{\mathbf{h}}(\mathbf{p}) = \text{G-MLP}(\bar{V}(\mathbf{p}), \gamma(\mathbf{p}))$, where $\bar{V}(\mathbf{p})$ is ray features aggregated and augmented from multi-view features, and $\gamma(\mathbf{p})$ is position encoding [S4]. Color is computed as: $\mathbf{c}(\mathbf{p}, \mathbf{d}) = \text{C-MLP}(\hat{\mathbf{h}}(\mathbf{p}), \mathbf{d})$ [S4]. The augmented feature $\bar{V}$ is a concatenation of $\{V_{avg}, V_{var}, \text{RGB}_{avg}, \text{RGB}_{var}\}$ [S4]. Grid resolution for 3D detection is $40 \times 40 \times 16$ [S4]. Source image feature map resolution is typically $160 \times 120$ [S4].

### Feature Sampling from Multi-View 2D Features
Sampling point features along the ray from multi-view image features instead of low-resolution volume features to address the need for high-resolution images in NeRF [S4]. Sampling from image features achieves 20.51 PSNR for novel-view synthesis compared to 18.93 with volume sampling [S4].

## Background
Feature Fields generalize the NeRF formulation to predict arbitrary per-point feature signals rather than only color [S1, unofficial]. Semantic-NeRF was notably the first to introduce semantic fields within the NeRF framework, using ground-truth semantic segmentation maps to augment the radiance field with per-point semantic logits [S1, unofficial]. N3F, LERF, and ISRF also learn additional feature fields to realize self-supervised 3D segmentation [S3]. With the emergence of the Segment Anything Model (SAM), methods like SA3D generate initial 2D masks with prompts and iteratively refine 3D masks [S3]. Feature 3DGS is the first method for feature field distillation based on 3D Gaussian Splatting [S3].

## Sources
[S1] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
[S2] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
[S3] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
[S4] NeRF-Det: Learning Geometry-Aware Volumetric Representation for Multi-View 3D Object Detection, https://doi.org/10.1109/iccv51070.2023.02131, fetched 2026-10-30
