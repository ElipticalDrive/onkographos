# Binary and Soft Classification

Also called: Hard and Soft Classification, Thresholding and Ramp Classification

## Overview
Binary and Soft Classification is the foundational family of volume rendering methods that map scalar voxel intensities to optical properties (opacity and color) to determine visibility. It solves the problem of selecting which data contributes to the final image by defining inclusion criteria, ranging from discrete hard thresholds to continuous smooth transitions. It sets itself apart from advanced siblings like [Probabilistic and Statistical Modeling](5932-probabilistic-and-statistical-modeling.md) by relying on deterministic, user-defined or simple heuristic mappings rather than statistical distributions, and from [Data Feature-Based Classification](5934-data-feature-based-classification.md) by typically operating on raw scalar values rather than complex derived feature spaces.

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input to the classification function is typically a scalar value $v = f(\mathbf{x})$ representing the voxel intensity or density at position $\mathbf{x}$ [S1, S2, S3, unofficial][S4]. Advanced methods extend this domain to include derived features such as gradient magnitude $\|\nabla f(\mathbf{x})\|$, curvature, or texture metrics [S2, S5, unofficial].

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

### [Piecewise-Linear Classification](881-piecewise-linear-classification.md)
A local classification technique that maps voxel intensity and derived features (such as gradient magnitude) to RGBA optical properties using piecewise-linear functions, commonly called ramps. It solves the problem of emphasizing specific data features by allowing users to define smooth transitions between material classes, distinguishing it from the hard thresholds of [Binary Classification](931-binary-classification.md) and enabling multi-channel control unlike basic single-channel methods. It is a specific instance of [Binary and Soft Classification](6229-binary-and-soft-classification.md) that relies on linear interpolation between user-defined keypoints. See [Piecewise-Linear Classification](881-piecewise-linear-classification.md).

### [Binary Classification](931-binary-classification.md)
A volume rendering method that reduces memory consumption and accelerates rendering by categorizing input volume voxels into two distinct groups: "1-voxels" (included) and "0-voxels" (discarded) based on a threshold criterion. Unlike soft or piecewise-linear methods that assign continuous opacity values, this technique makes discrete "yes" or "no" decisions, creating sharp transitions in the output. It sets itself apart from siblings like [Piecewise-Linear Classification](881-piecewise-linear-classification.md) by eliminating intermediate opacity values, thereby enabling significant speedups through voxel culling, albeit at the cost of introducing high-frequency spectral artifacts. See [Binary Classification](931-binary-classification.md).

### [Multiple Material Classification](6343-multiple-material-classification.md)
Extends single-material classification techniques to identify and render distinct substances within a volume simultaneously, allowing users to assign specific opacity and color properties to different classes (e.g., high opacity for an organ, low opacity for context). It differs from [Binary Classification](931-binary-classification.md) by supporting more than two distinct material definitions and from [Piecewise-Linear Classification](881-piecewise-linear-classification.md) by managing multiple independent classification models rather than a single unified transfer function. See [Multiple Material Classification](6343-multiple-material-classification.md).

### Non-Binary Classifier
A CRIO process that smoothly transitions from 'no' to 'yes' with varying degrees of acceptance, generating soft edges [S1, unofficial]. It applies smooth transitions between regions of interest and no interest, avoiding sharp corners in the classifier function [S1, unofficial]. Unlike binary classifiers, it introduces significantly fewer high frequencies, reducing aliasing artifacts [S1, unofficial].

## Background
The distinction between binary and non-binary (soft) classification was formalized in early volume rendering research to address spectral artifacts. Binary classifiers introduce sharp transitions in the output function, which introduces high frequencies in the output spectrum, leading to visible rendering artifacts [S1, unofficial]. Non-binary classifiers were developed to avoid these sharp corners, generating soft edges and significantly fewer high frequencies [S1, unofficial]. This foundational work is documented in the University of North Carolina technical report on parallel feed-forward volume rendering algorithms [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
