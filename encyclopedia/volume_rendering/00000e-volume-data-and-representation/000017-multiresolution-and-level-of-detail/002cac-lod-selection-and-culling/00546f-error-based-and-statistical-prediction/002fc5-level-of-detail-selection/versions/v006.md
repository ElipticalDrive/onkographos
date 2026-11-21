# Level-of-Detail Selection
Also called: Image-based LOD selection

## Overview
Level-of-Detail Selection is an algorithm that uses an image-based quality metric to select appropriate levels of detail for rendering large volumetric data, thereby improving image quality during real-time rendering [S1]. It differs from its sibling [Histogram Spectra for LOD Selection](11687-histogram-spectra-for-lod-selection.md) by relying on direct image fidelity metrics rather than precomputed statistical histogram spectra to drive the selection decision.

## How it works
The method operates within a GPU-based raycasting pipeline to handle large volumetric datasets [S1].

**Inputs and Outputs:**
*   **Input:** Large volumetric data; current rendering parameters (viewing distance, resolution).
*   **Output:** Selected LOD index for each subvolume or tile.

**Data Structures:**
Not given in the sources.

**Steps:**
1.  **Preprocessing:** The volumetric data is prepared for block transformation coding, a technique previously proposed by Fout and Ma for GPU-based real-time volume compression and decompression [S1].
2.  **Metric Calculation:** An image-based quality metric is computed to evaluate the visual difference between the current LOD and a reference (higher fidelity) LOD [S1].
3.  **Selection:** The LOD is selected based on the quality metric to ensure the rendered image meets a quality threshold while optimizing for performance [S1].

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### Residual-Proxy Adaptive Selector
An adaptive LOD selection method that uses residual proxies to recover quality over fixed threshold methods at comparable primitive counts [S2, unofficial]. On the 'garden' scene, it recovers 0.49 dB over the fixed threshold at a comparable primitive count (1.05x) [S2, unofficial]. On city-scale models, it exhibits a wider near-lossless window because the aerial model is already tessellated at its training viewing distance, exhausting sub-pixel redundancy at small compression factors ($c$), leaving surplus in super-pixel collapse [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method was developed by Wang et al. to provide interactive volume renditions for large volumetric data on the GPU [S1]. It builds on earlier work by Fout and Ma regarding block transformation coding for volume compression [S1]. It is used in real-time volume rendering pipelines where image quality must be maintained while managing computational load [S1].

## Sources
[S1] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S2] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
