# Color Composition
Also called: AWP Color Composition, Adaptive Weighted Color Composition

## Overview
Color composition aggregates multiple color samples along a ray into a final pixel color using learned or adaptive weights [S1]. It solves the problem of combining contributions from different semantic or physical regions (e.g., rigid vs. non-rigid motion) into a single coherent visual output [S1]. It sets itself apart from sibling methods like [Windowing](6611-windowing.md) and [Colormap Application for Visualization](7714-colormap-application-for-visualization.md) by explicitly modeling weighted blending of distinct color sources rather than simple thresholding or direct mapping [S1].

## How it works
The method varies by variant. The primary implementation (AWP Color Composition) uses a neural network to predict weights [S1].

### Inputs and Outputs
**Inputs:**
*   $\hat{C}_{s;i}$: Predicted or sampled colors for sample $i$ along ray $s$.
*   $ms_{;0,...,k}$: Rigid weights.
*   $m^{p-rig}_{s;i}$: Adaptive non-rigid weights.
*   $\tilde{m}^{p}_{s;0}$: Adaptive weight for the primary component.

**Outputs:**
*   $\hat{B}_{sp}$: Composed color for rigid components.
*   $\tilde{B}_{sp}$: Composed color for adaptive components.

### Data Structures
*   **Weights:** Scalars normalized such that $\sum_{i=0}^{k} ms_{;i} = 1$ [S1].
*   **Colors:** RGB vectors.

### Steps and Formulas

#### 1. Tone Mapping
Before composition, all input colors are tone-mapped using a gamma function [S1]:
$$ g(c_0) = c_0^{\frac{1}{2.2}} $$
where $c_0$ is the predicted scene irradiance or color value [S1].

#### 2. Weighted Composition (AWP Variant)
The final color is computed as a sum of tone-mapped colors weighted by rigid and adaptive factors [S1]:

$$ \hat{B}_{sp} = ms_{;0} g(\hat{C}_{s;0}) + \sum_{i=1}^{k} m^{p-rig}_{s;i} g(\hat{C}_{s;i}) $$

$$ \tilde{B}_{sp} = \tilde{m}^{p}_{s;0} g(\hat{C}_{s;0}) + \sum_{i=1}^{k} \tilde{m}^{p-rig}_{s;i} g(\hat{C}_{s;i}) $$

Where:
*   $ms_{;0,...,k}$ are rigid weights normalized along the motion axis $k$ [S1].
*   $m^{p-rig}_{s;i}$ and $\tilde{m}^{p-rig}_{s;i}$ are adaptive weights derived from an Adaptive Weighted Prediction (AWP) module [S1].
*   $k$ is a hyper-parameter controlling the number of rigid camera motions [S1].

#### 3. CTF Variant
Details for a CTF-specific blending variant are not given in the sources.

### Parameters
*   $k$: Number of rigid camera motions (hyper-parameter) [S1].

### Edge Cases and Pitfalls
*   **Normalization:** Rigid weights $ms_{;0,...,k}$ must be normalized along the motion axis $k$ to ensure $\sum_{i=0}^{k} ms_{;i} = 1$ [S1].
*   **Tone Mapping:** Failure to tone-map predicted irradiance colors before composition can lead to incorrect visual results [S1].
*   **Transparency:** The transparency compositing formula (see Subtopics) is often omitted in basic raycasters, resulting in opaque images only [S2, unofficial].

## Subtopics

### [Lightness-Controlled Color Harmonization](10770-lightness-controlled-color-harmonization.md)
Lightness-Controlled Color Harmonization is a technique for volume visualization that transforms a transfer function’s color distribution to fit a specific harmonic template while strictly preserving the original lightness values. It solves the problem that standard hue-based harmonization alters lightness, thereby destroying the perceptual contrast and "pop-out" effects critical for highlighting density intervals. It sets itself apart from [Windowing](6611-windowing.md) and [Colormap Application for Visualization](7714-colormap-application-for-visualization.md) by operating in the perceptually uniform CIE LAB space to decouple hue shifts from lightness changes, ensuring that visual salience is maintained during colorization. Choose this when perceptual contrast must be preserved during colorization. [Lightness-Controlled Color Harmonization](10770-lightness-controlled-color-harmonization.md)

### [Harmonized Color Change](14285-harmonized-color-change.md)
Harmonized Color Change is a local feature enhancement technique for volume rendering that replaces the computed color of a sample with its harmonic (complementary) counterpart when the sample represents a significant local density peak. It solves the problem of subtle internal structures being lost in the global color palette by providing high-contrast visual cues for boundaries and high-density regions without altering the overall perceptual impression of the volume. It differs from [Lightness-Controlled Color Harmonization](10770-lightness-controlled-color-harmonization.md) by using hue complementarity rather than lightness adjustment, and from [Iterative Color Matching](12841-iterative-color-matching.md) by being a direct, non-iterative replacement based on local density gradients. Choose this for highlighting local density peaks without global palette shifts. [Harmonized Color Change](14285-harmonized-color-change.md)

### [CTF Color Blending](14808-ctf-color-blending.md)
Details for this variant are not given in the sources. Choose this when semantic clustering is required alongside voxel data. [CTF Color Blending](14808-ctf-color-blending.md)

### Transparency Compositing Formula
Applies the standard alpha compositing formula recursively along the ray:
$$ C_{out} = C_{in} (1 - \alpha(X_i)) + C(X_i) \alpha(X_i) $$
Where $C(X_i)$ is shading from the local gradient, $\alpha(X_i)$ is opacity from the CT value, $C_{out}$ is outgoing intensity, and $C_{in}$ is incoming intensity [S2, unofficial]. The recursion stops when $\alpha(X_i) = 0$ (e.g., bone) [S2, unofficial]. **Pitfall:** Many implementations (including the one in [S2, unofficial]) do not implement this formula, offering only opaque images [S2, unofficial].

## Operating characteristics
*   **Complexity:** Linear in the number of samples $k$ and ray length. The AWP variant adds neural network inference cost for weight prediction [S1].
*   **Performance:** Dependent on the efficiency of the MLPs used for weight prediction in AWP [S1].
*   **Strengths:** Allows for physically plausible blending of motion-blurred or semantically distinct regions [S1].
*   **Limitations:** Requires careful normalization of weights [S1]. The transparency formula is often omitted in simple implementations [S2, unofficial].

## Background
The AWP Color Composition method originates from DP-NeRF, which introduces adaptive weighted prediction for deblurring Neural Radiance Fields [S1]. The transparency compositing approach is a standard volume rendering technique, though specific implementations vary [S2, unofficial].

## Sources
[S1] DP-NeRF: Deblurred Neural Radiance Field with Physical Scene Priors, https://arxiv.org/abs/2211.12046v4, fetched 2026-10-26
[S2] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
