# Spatial and Localized TFs

Also called: Spatial Transfer Functions, Localized Transfer Functions

## Overview
This family encompasses methods where the classification function $C(\mathbf{x})$ depends explicitly on the spatial position $\mathbf{x}$ in addition to the data value, allowing different regions of the volume to be rendered with distinct visual styles or transfer function parameters. It solves the limitation of global transfer functions, which apply the same mapping to the entire volume, by enabling spatially varying opacity, color, or illustrative effects [S1, book]. It differs from standard global TFs by introducing spatial priors (geometric zones or anatomical models) to modulate the classification, and from sibling methods like [2D Transfer Function with Gradient Magnitude](5491-2d-transfer-function-with-gradient-magnitude.md) by using explicit spatial coordinates rather than derived data features for localization.

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input to the classification function is typically a scalar value $v = f(\mathbf{x})$ representing the voxel intensity or density at position $\mathbf{x}$ [S1, book][S2, unofficial][S4]. Advanced methods extend this domain to include derived features such as gradient magnitude $\|\nabla f(\mathbf{x})\|$, curvature, or texture metrics [S2, unofficial][S5, unofficial].

**Output Range:**
The output is a set of optical properties, most commonly:
*   **Opacity ($\alpha$):** A value in $[0, 1]$ representing the probability of absorption or the inverse of transparency. $\alpha=0$ is fully transparent; $\alpha=1$ is fully opaque [S5, unofficial].
*   **Color ($C$):** A vector $(r, g, b)$ representing the emitted or reflected light color [S2, unofficial][S4].
*   **Weight:** In some emission-only models, a weight $w$ is assigned to modulate the contribution of samples during accumulation [S6, unofficial].

**Classification Types:**
1.  **Binary Classification:** Uses hard thresholds to include or exclude data [6229-binary-and-soft-classification.md]. Values below a threshold are assigned $\alpha=0$, and values above are assigned $\alpha=1$ (or a constant high value) [G].
2.  **Soft Classification:** Uses smooth transitions (e.g., piecewise linear ramps, Gaussian bumps, or power functions) to assign intermediate opacity values [S3, unofficial][S8, unofficial].

**Evaluation Order:**
The timing of classification relative to sampling interpolation defines two main approaches [S2, unofficial][S5, unofficial]:
*   **Pre-classification:** The transfer function is applied to scalar values at the grid points (voxels) before sampling. Colors and opacities are interpolated during ray casting. Pre-classification produces blurred images when the image resolution is higher than the volume resolution, but prevents color bleeding if color interpolation is done with opacity-weighted colors [S2, unofficial].
*   **Post-classification:** Scalar values are interpolated from the volume texture during ray casting, and the transfer function is applied to these continuous sample values. Post-classification produces images with defined edges but is prone to aliasing because high-frequency details may be missed [S2, unofficial].

**Implementation Structures:**
Transfer functions are commonly implemented as piecewise linear functions defined by user-specified keypoints [S3, unofficial], lookup tables (LUTs) stored as 1D, 2D, or 3D textures for fast GPU sampling [S3, unofficial][S6, unofficial], or shader functions evaluated dynamically during ray casting [S5, unofficial].

## Subtopics

### [Spatial Style Zones](6652-spatial-style-zones.md)
[Spatial Style Zones](6652-spatial-style-zones.md) is a technique for volume illustration where regions of volume space (zones) are assigned distinct rendering styles, defined as sets of transfer functions controlling appearance per material. It solves the problem of managing complex, multi-parameter transfer functions by allowing users to apply different visual effects (e.g., sketch, illumination) to specific spatial regions independently. It differs from Model-Based Local Transfer Functions by defining zones via simple geometric primitives (ellipsoids or rectangles) rather than model-derived features, and from standard spatial TFs by associating entire style sets (combinations of effects) with zones rather than single scalar mappings.

### [Model-Based Local Transfer Functions](7281-model-based-local-transfer-functions.md)
[Model-Based Local Transfer Functions](7281-model-based-local-transfer-functions.md) is a visualization method for medical image volumes that applies multiple voxel-specific transfer functions simultaneously to different anatomical regions, determined by a deformable anatomical model registered to the volume. It solves the problem of reviewing complex medical data by allowing a radiologist to efficiently visualize disjoint anatomical regions with their optimal transfer functions in a single pass, rather than switching between global settings. It differs from sibling methods like Spatial Style Zones by explicitly using a deformable anatomical model (via Morphon registration) to define the spatial priors for the transfer functions, ensuring the classification boundaries align with tissue anatomy rather than arbitrary spatial coordinates.

### Spatial Transfer Functions
**Definition:**
A function that defines a geometrical transformation of the scalar values of volumetric models to allow effects such as splitting, squeezing, and sweeping, handling discontinuities in the rendering method [S1, book].

**Steps:**
1.  The volumetric model with the spatial transfer function is rendered by first computing the transformation on the bounding volume [S1, book].
2.  Map back to find the actual volumetric values [S1, book].
3.  Composite the values using a standard ray casting approach [S1, book].

### Diagonal Busyness Factor (DF)
**Definition:**
A texture probe used to clarify ambiguous situations by relaying senses such as regularity, coarseness, fitness, directionality, and granularity. It helps distinguish objects from backgrounds with similar colors but different textures (e.g., hatches of a chair vs. brown background) [S2, unofficial].

**Steps:**
1.  Calculate DF as the sum of color differences among the central pixel and diagonal pixels [S2, unofficial].

**Parameters:**
*   $C$: The central pixel.
*   $x$: The diagonal pixels [S2, unofficial].

## Background
Spatial transfer functions were introduced to support illustrative visualization and non-physically based manipulations of volumetric data, such as surgical-like cuts, peeling, moving, sweeping, and unrolling [S1, book]. These methods allow for handling discontinuities in rendering that direct data manipulation methodologies struggle with [S1, book]. The concept of using spatial priors for classification has evolved to include model-based approaches in medical imaging, where deformable models (e.g., Morphon registration) are used to align anatomical structures with patient-specific data, enabling localized transfer functions that respect tissue boundaries [S1, book]. The Diagonal Busyness Factor was developed in the context of segmentation-assisted object distinction for direct volume rendering, utilizing texture properties to resolve ambiguities between objects and backgrounds [S2, unofficial].

## Sources
[S1] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
[S2] Segmentation Assisted Object Distinction for Direct Volume Rendering, https://doi.org/10.22044/jadm.2019.7207.1854, fetched 2026-10-29 (unofficial)
