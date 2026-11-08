# Importance-Driven Visualization Model

Also called: Importance-Driven Visualization

## Overview
The Importance-Driven Visualization Model is a framework that dynamically adjusts visual representation based on computed feature importance to resolve occlusion and maximize perceptible information [S1, unofficial]. It replaces viewpoint-independent classification with a pipeline of importance specification, compositing, and sparseness representation [S1, unofficial]. It differs from standard volume rendering by decoupling the visibility of features from their scalar values, allowing less important structures to be suppressed regardless of their intensity, thereby guiding viewer attention to salient boundaries and regions [S1, unofficial].

## Shared principles
The framework operates on the principle that visual clarity is improved when less important features are made transparent or desaturated, while important features remain opaque and saturated [S1, unofficial]. The core mechanism involves three stages:
1.  **Importance Specification:** Computing a scalar importance value for each voxel or region. This value drives the representation density, opacity, and color saturation [S1, unofficial].
2.  **Compositing:** Integrating the importance values into the rendering pipeline, typically during raycasting, to modulate the contribution of samples [S1, unofficial].
3.  **Sparseness Representation:** Adjusting the visual density of the representation (e.g., opacity, color saturation) based on the importance level. High importance results in opaque, saturated colors, while low importance results in transparent, desaturated colors [S1, unofficial].

The importance value is a scalar in the range $[0, \text{max}]$ [S1, unofficial]. Typical visualization levels use specific importance thresholds (e.g., 0, 0.1, 0.7, max) to define distinct layers of sparseness [S1, unofficial]. The model supports automatic specification of importance based on feature classification or user-steered assignment to segmented objects [S1, unofficial].

## Subtopics

### [Saliency-based Opacity Modulation](7041-saliency-based-opacity-modulation.md)
Saliency-based Opacity Modulation is an illustrative volume rendering technique that dynamically adjusts voxel opacity based on computed spatial importance (saliency) values to guide viewer attention toward salient features, such as boundaries, while suppressing less relevant internal structures. It differs from standard volume rendering by replacing fixed transfer functions with a data-driven modulation pipeline that enhances perceptible information through thresholding and scaling of opacity based on local intensity gradients. See [Saliency-based Opacity Modulation](7041-saliency-based-opacity-modulation.md).

### [Semantic Shader Augmentation](7401-semantic-shader-augmentation.md)
Semantic Shader Augmentation is a technique that automatically extends an input GPU shader program with rule-based rendering functionality, replacing static visualization mappings (such as fixed transfer functions) with dynamic mappings defined by semantic rules and fuzzy membership functions. It solves the problem of rigid, hard-coded semantic layers by allowing arbitrary existing shaders to be dynamically augmented with the capability to evaluate illustration rules, enabling domain experts to interactively explore semantic mappings without programming skills. Unlike the original semantic-layers approach where rule-based functionality is hard-coded a priori, or UberShader/SuperShader which combine pre-coded fragments, this method generates code fragments entirely on the fly, allowing minimal overhead and quick exploration of different semantic mappings for various visualization techniques. See [Semantic Shader Augmentation](7401-semantic-shader-augmentation.md).

### Intent-Based Illustration Generation
A system that attempts to predict what kind of illustration a viewer would like to see by using an intent-based approach borrowing from AI knowledge bases [S2, unofficial]. It defines rules that describe communicative intent and uses stylistic decisions that reflect particular goals to generate the actual illustrations [S2, unofficial].

### Voxel Saliency
A measure of regional importance for voxels, defined as the magnitude of the gradient of the obscurance field. It highlights edges, occlusion variations, and corners [S3]. The saliency $S(p)$ for a voxel $p$ is computed as:
$$ S(p) = \sqrt{A^2 + B^2 + C^2} $$
where $A, B, C$ are components of the obscurance gradient estimated by 4D linear regression [S3]:
$$ A = \sum_k w(k)O(k)x(k), \quad B = \sum_k w(k)O(k)y(k), \quad C = \sum_k w(k)O(k)z(k) $$
Here, $k$ iterates over voxels in a $5^3$ neighborhood centered at $p$, $w(k)$ is the distance between $p$ and $k$, $O(k)$ is the obscurance of voxel $k$, and $x(k), y(k), z(k)$ are the components of the vector from $p$ to $k$ [S3]. The resulting saliency is scaled to $[0, 1]$ [S3]. It is used to select the best viewpoints by maximizing visible saliency and to enhance visualization by modifying the transfer function (opacity modulation) based on saliency [S3].

### Stippling Modulation via Spatial Importance
Regulation of dot density in stipple rendering based on the normalized absolute spatial importance map, where higher importance leads to denser dots or white regions [S4, unofficial]. The density of the dots is modulated by the spatial importance map [S4, unofficial]. Two user-defined parameters are used: a threshold $t$ above which everything is rendered white, and a factor $f$ that regulates the density scale [S4, unofficial]. Additionally, a random value $r(s)$ is generated for each sample $s$ in the ray casting [S4, unofficial].

## Background
The Importance-Driven Visualization Model was introduced by Ivan Viola and M. Eduard Gröller [S1, unofficial]. It was implemented in the VolumeShop tool for interactive direct volume illustration, supporting multi-object rendering, 3D painting segmentation, stylized rendering, smart visibility, and labeling [S1, unofficial]. The model addresses the problem of occlusion in 3D visualization, where important features are often hidden by less important ones [S1, unofficial]. Traditional techniques classify features independently of the viewpoint, limiting the ability to reveal important structures [S1, unofficial]. The model allows for automatic specification of importance based on feature classification or user-steered assignment [S1, unofficial].

## Sources
[S1] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
[S2] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S3] Obscurance-based Volume Rendering Framework, https://doi.org/10.2312/vg/vg-pbg08/113-120, fetched 2026-10-26
[S4] Interactive Volume Illustration Using Intensity Filtering, https://doi.org/10.2312/compaesth/compaesth10/051-058, fetched 2026-10-26 (unofficial)
