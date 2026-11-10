# Texture-based TF (Histogram Statistics)

Also called: Run-length/Co-occurrence TF, Statistical TF (Texture)

## Overview
This method classifies volumetric data by analyzing local textural properties using histogram statistics, run-length, and co-occurrence matrices to capture geometrical and structural properties [S1, unofficial]. It solves the problem of discriminating structures that have similar intensity values but different internal textures [S1, unofficial]. Unlike [Spatial Style Zones](6652-spatial-style-zones.md) which rely on spatial position, or [Model-Based Local Transfer Functions](7281-model-based-local-transfer-functions.md) which rely on shape models, this method relies on statistical descriptors of intensity distributions within local neighborhoods [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** A scalar volume dataset $V$.
**Outputs:** A multidimensional transfer function mapping local texture metrics to optical properties (opacity $\alpha$ and color $C$).

**Data Structures:**
The method pre-computes a set of statistical metrics for each voxel. The total number of pre-computable metrics is 20 [S1, unofficial]. These metrics are typically stored in a texture or lookup table structure compatible with the rendering pipeline.

**Steps:**
1.  **Partitioning:** Divide the dataset into overlapping sub-volumes to keep data size and computation times under control [S1, unofficial].
2.  **Metric Extraction:** For each sub-volume (or local neighborhood centered at voxel $\mathbf{x}$), compute the following textural properties:
    *   **First-order statistics:** Variance, skewness, kurtosis, and occurrence of predefined intensity values [S1, unofficial].
    *   **Second-order statistics:** Likelihood of observing two different intensity values at a certain distance (derived from co-occurrence matrices) [S1, unofficial].
    *   **Run-length statistics:** Properties derived from run-length matrices [S1, unofficial].
3.  **Classification:** Map the computed vector of metrics to opacity $\alpha \in [0, 1]$ and color $C \in [0, 1]^3$ using a user-defined multidimensional transfer function [S1, unofficial].
4.  **Rendering:** Use the resulting optical properties during ray casting [S1, unofficial].

**Parameters:**
*   **Number of metrics:** 20 [S1, unofficial].
*   **Sub-volume size:** Not given in the sources.
*   **Overlap:** Not given in the sources.

**Edge cases and pitfalls:**
*   The method works on fixed-size data extraction regions, precluding it from differentiating between homogenous and inhomogeneous regions in a dataset [S1, unofficial].
*   Computation time can be high due to the extraction of multiple statistical metrics; however, modern GPUs reduce this time significantly [S1, unofficial].

## Subtopics

### Texture Attribute
A composite property computed by local statistical descriptors, originating from pattern recognition [S2, unofficial]. It is not formed by a fixed number of attributes, distinguishing it from the specific 20-metric histogram approach [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** High, due to the computation of multiple statistical metrics (variance, skewness, kurtosis, co-occurrence, run-length) for overlapping sub-volumes [S1, unofficial].
*   **Memory Complexity:** Requires storage for the pre-computed metrics (20 channels per voxel) and the multidimensional transfer function [S1, unofficial].
*   **Strengths:** Effective at discriminating structures with similar intensity values but different textures [S1, unofficial].
*   **Limitations:** Cannot differentiate between homogenous and inhomogeneous regions within the fixed extraction region [S1, unofficial].
*   **When to use:** When intensity-based classification fails to separate structurally distinct tissues or materials [S1, unofficial].

## Background
This method was described in reference [20] of [S1, unofficial], which proposes an approach based on analyzing local textural properties of the volume. It captures both geometrical and structural properties by using histogram statistics, run-length, and co-occurrence matrices [S1, unofficial]. The concept of texture as a composite property computed by local statistical descriptors originates from pattern recognition [S2, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S2] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
