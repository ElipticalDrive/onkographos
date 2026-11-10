# Screen-Door Transparency with Importance Compositing

Also called: Screen-Door Transparency, Importance Compositing

## Overview
Screen-Door Transparency with Importance Compositing is a rendering technique that connects feature importance to spatial sampling density using screen-door transparency [S1, unofficial]. It solves the problem of occluding important features by less important ones by dynamically adjusting sampling density and opacity based on feature importance and occlusion relationships, rather than using a fixed global transfer function [S1, unofficial]. Unlike Maximum Importance Projection (MImP), which renders only the feature with the highest importance along the ray, this method composites multiple features to preserve context while suppressing occluding less important structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Volumetric dataset, feature importance values for each voxel/feature, viewing ray.
- Output: Rendered image with varying density and transparency based on importance.

**Data Structures:**
- Importance map: Scalar field where each voxel has an importance value $I \in [0, 1]$.
- Grid spacing: Variable step size along rays, inversely proportional to importance.

**Steps:**
1. **Determine Importance:** For each ray, determine the importance of intersected features. High importance features are opaque and color saturated; low importance features are transparent and desaturated [S1, unofficial].
2. **Assign Grid Spacing:** Assign grid spacing inversely proportional to importance. High importance uses small grid spacing (dense sampling); low importance uses big grid spacing (sparse sampling) [S1, unofficial].
3. **Render Focus Features:** Render focus features (high importance) with dense sampling to ensure they are clearly visible [S1, unofficial].
4. **Render Context Features:** Render context features (lower importance) with adaptive sampling:
   - If context features occlude focus features, use sparse sampling to suppress them [S1, unofficial].
   - Otherwise, use dense sampling to preserve context [S1, unofficial].

**Screen-Door Transparency Strategy:**
- Determine the distribution of visible voxels such that it is uniform [S2, unofficial].
- Form a wireframe-like structure with the visible voxels, simulating transparency by distributing visible voxels uniformly [S2, unofficial].

**Formulas:**
- Grid spacing $s$ for a voxel with importance $I$: $s \propto \frac{1}{I}$ [S1, unofficial].
- Opacity $\alpha$ for a voxel with importance $I$: $\alpha$ is high for high $I$, low for low $I$ [S1, unofficial].

**Parameters:**
- Importance threshold: Not given in the sources.
- Sparseness factor: Not given in the sources.

**Edge Cases and Pitfalls:**
- Degenerate input: If importance values are uniform, the technique degrades to standard volume rendering.
- Precision: Sparse sampling may miss thin structures if the grid spacing is too large.

## Subtopics

### Screen-Door Transparency
A strategy to simulate transparency by distributing visible voxels uniformly to form a wireframe-like structure, determining which voxels are rendered based on the level of sparseness [S2, unofficial]. The distribution of visible voxels is uniform, and the impact of increasing sparseness is shown by reducing the volume to fewer strong iso-surfaces based on gradient and curvature magnitude [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of voxels sampled, which varies based on importance. High importance regions require more samples, increasing time complexity.
- **Memory Complexity:** Requires storage for importance map and volumetric data.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the resolution of the volumetric data and the complexity of the importance map.
- **Strengths:** Effectively resolves occlusion of important features while preserving context.
- **Limitations:** May miss thin structures if sparse sampling is too aggressive.
- **When to Use:** When it is critical to highlight important features without completely removing less important context.
- **When Not to Use:** When uniform detail is required across all features.

## Background
Originated by Ivan Viola and M. Eduard Gröller in the context of illustrative visualization for medicine and science [S1, unofficial]. It builds on techniques used in technical and medical illustrations, such as cutaway views and ghosted views, to resolve occlusion of prominent information [S1, unofficial].

## Sources
[S1] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
[S2] (PDF) Importance-Driven Volume Rendering, https://www.researchgate.net/publication/220943935_Importance-Driven_Volume_Rendering, fetched 2026-11-03 (unofficial)
