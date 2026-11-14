# Harmonized Color Change

Also called: Harmonic color feature emphasis, Complementary color replacement

## Overview
Harmonized Color Change is a local feature enhancement technique for volume rendering that replaces the computed color of a sample with its harmonic (complementary) counterpart when the sample represents a significant local density peak [S1, unofficial]. It solves the problem of subtle internal structures being lost in the global color palette by providing high-contrast visual cues for boundaries and high-density regions without altering the overall perceptual impression of the volume [S1, unofficial]. It differs from [Lightness-Controlled Color Harmonization](10770-lightness-controlled-color-harmonization.md) by using hue complementarity rather than lightness adjustment, and from [Iterative Color Matching](12841-iterative-color-matching.md) by being a direct, non-iterative replacement based on local density gradients.

## How it works
**Inputs:**
- Volume dataset $V$ with scalar density values.
- Ray casting parameters (view direction, sample positions).
- Base color transfer function $C_{base}(d)$ mapping density $d$ to RGB.
- Shading model (e.g., Phong) to compute initial sample color $C_{phong}$.
- Neighborhood radius $r$ for density averaging.
- Threshold factor $\alpha$ for density comparison (Not given in the sources).

**Outputs:**
- Final pixel color $C_{final}$.

**Data Structures:**
- Sample point $p$ with position $\mathbf{x}$, density $d(\mathbf{x})$, and normal $\mathbf{n}$ (if shading is used).
- Neighborhood set $N(\mathbf{x}) = \{ \mathbf{x}_i \mid \|\mathbf{x}_i - \mathbf{x}\| \le r \}$.

**Steps:**
1. **Compute Base Color:** Calculate the initial color $C_{phong}$ for the sample at $\mathbf{x}$ using the chosen shading model (e.g., Phong) and the base transfer function [S1, unofficial].
2. **Determine Harmonic Color:** Convert $C_{phong}$ to a hue-based representation (e.g., HSV or HSL). Identify the complementary hue $H_{comp}$ by rotating the hue angle by $180^\circ$ on the color circle. Reconstruct the harmonic color $C_{harmonic}$ using $H_{comp}$ and the original saturation/value components [S1, unofficial].
3. **Check Density Condition:** Calculate the average density $\bar{d}$ of the neighborhood $N(\mathbf{x})$. Check if the sample's density $d(\mathbf{x})$ is significantly larger than $\bar{d}$ [S1, unofficial].
   - Condition: $d(\mathbf{x}) > \bar{d} + \alpha$ or $d(\mathbf{x}) > k \cdot \bar{d}$ (exact formula not given in the sources).
4. **Replace and Enhance:** If the condition is met, replace $C_{phong}$ with $C_{harmonic}$. Apply a lightening unsharp masking equation to emphasize the feature [S1, unofficial].
   - Formula: $C_{final} = C_{harmonic} + \beta (C_{harmonic} - C_{blurred})$ (Equation 4.2 in [S1, unofficial], specific coefficients not given in the sources).
   - If the condition is not met, $C_{final} = C_{phong}$ [S1, unofficial].

**Parameters:**
- Neighborhood radius $r$: Depends on the multi-scale volume hierarchy levels (2, 3, 4, 5, 6, 8, 16, 32) [S1, unofficial].
- Threshold factor $\alpha$: Not given in the sources.
- Lightening factor $\beta$: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Conversion between RGB and hue spaces must be handled carefully to avoid artifacts at hue boundaries (e.g., red-purple transition) [S1, unofficial].
- **Boundaries:** At volume boundaries, the neighborhood $N(\mathbf{x})$ may be incomplete, affecting $\bar{d}$ calculation. Not given in the sources.
- **Degenerate Input:** If the volume has uniform density, the condition $d(\mathbf{x}) > \bar{d}$ will never be met, and no enhancement occurs [S1, unofficial].

## Subtopics

### Contextual Color Ghosting
Contextual Color Ghosting is a variant used in focus+context rendering where the context (non-focus) data is rendered in grayscale to enhance the perception of the colored focus by contrast [S2, unofficial]. It is chosen when the user wants to reduce visual clutter in the background while preserving color information in the region of interest [S2, unofficial]. It differs from standard Harmonized Color Change by applying grayscale mapping to entire contextual clusters rather than individual samples based on local density peaks [S2, unofficial]. The color of the contextual cluster can be computed using structural coloring or, if the color ghosting flag is activated, set to grayscale by considering only the node's intensity rather than RGB values [S2, unofficial]. The opacity of contextual clusters is modulated inversely proportional to the depth of the nodes in the hierarchy [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot r^3)$ per sample, where $N$ is the number of samples and $r$ is the neighborhood radius, due to the density averaging step [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing sample colors and neighborhood data [S1, unofficial].
- **Measured Performance:** Tested on volume models up to $512 \times 512 \times 512$ voxels [S1, unofficial].
- **Scaling:** Performance scales with the resolution of the volume and the size of the neighborhood radius [S1, unofficial].
- **Strengths:** Enhances local features without changing the overall color impression; effective for pollen grain and orange data sets [S1, unofficial].
- **Limitations:** Requires careful tuning of neighborhood radius and threshold parameters; may introduce artifacts if the neighborhood is too small or large [S1, unofficial].
- **When to use:** When local density peaks need to be highlighted without altering the global color scheme [S1, unofficial].
- **When not to use:** When the volume has uniform density or when global color consistency is more important than local feature emphasis [S1, unofficial].

## Background
The method was proposed in the context of enhancing local features in volume visualization using harmonic colors to stress important details [S1, unofficial]. It was tested on various volume models, including pollen grains and orange data sets, demonstrating its effectiveness in emphasizing boundaries and high-density regions [S1, unofficial]. The approach is part of a broader effort to improve the perceptual quality of volume rendering by leveraging color theory principles [S1, unofficial].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S2] Structural focus+context rendering of multiclassified volume data, http://hdl.handle.net/2117/86811, fetched 2026-10-26 (unofficial)
