# Texture-based TF (Histogram Statistics)

Also called: Run-length/Co-occurrence TF, Statistical TF (Texture)

## Overview
This method classifies volumetric data by analyzing local textural properties using histogram statistics, run-length, and co-occurrence matrices to capture geometrical and structural properties [S1, unofficial]. It solves the problem of discriminating structures that have similar intensity values but different internal textures [S1, unofficial]. It sets itself apart from its siblings, [Spatial Style Zones](6652-spatial-style-zones.md) and [Model-Based Local Transfer Functions](7281-model-based-local-transfer-functions.md), by computing fixed-size statistical descriptors over overlapping sub-volumes rather than relying on spatial style zones or model-based shape fitting [S1, unofficial][G].

## How it works
**Inputs:** A volumetric dataset $V$ [S1, unofficial].
**Outputs:** A multi-dimensional transfer function $C(\mathbf{x})$ mapping local texture metrics to optical properties [S1, unofficial][G].

**Data Structures:**
The method pre-computes a set of statistical metrics for each overlapping sub-volume [S1, unofficial]. The total number of pre-computable metrics is 20 [S1, unofficial]. These metrics are combined into the transfer function [S1, unofficial].

**Steps:**
1. **Partitioning:** Divide the dataset into overlapping sub-volumes to keep data size and computation times under control [S1, unofficial].
2. **Metric Extraction:** For each sub-volume, analyze textural properties [S1, unofficial]:
    * **First-order statistics:** Compute variance, skewness, and kurtosis of the intensity distribution within the sub-volume [S1, unofficial].
    * **Second-order statistics:** Compute the likelihood of observing two different intensity values at a certain distance (co-occurrence) [S1, unofficial].
    * **Run-length statistics:** Analyze the occurrence of predefined intensity values and their run lengths [S1, unofficial].
    * **Histogram statistics:** Analyze the occurrence of predefined intensity values [S1, unofficial].
3. **Classification:** Map the resulting feature vector to opacity and color using the transfer function [S1, unofficial].

**Parameters:**
* **Number of metrics:** 20 [S1, unofficial].
* **Sub-volume size:** Not given in the sources [S1, unofficial].
* **Overlap:** Not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
* The method works on fixed-size data extraction regions, which precludes it from differentiating between homogenous and inhomogeneous regions within a dataset [S1, unofficial].

## Subtopics

### Texture Attribute
A composite property computed by local statistical descriptors, originating from pattern recognition [S2, unofficial]. It is not formed by a fixed number of attributes, distinguishing it from the 20-metric approach described in the main method [S2, unofficial]. When to choose it: when a flexible, non-fixed set of statistical descriptors is preferred over a rigid feature space [G].

## Operating characteristics
* **Complexity:** Computation scales with the number of overlapping sub-volumes and the 20 pre-computed metrics per region [S1, unofficial].
* **Performance:** Tested on an nVidia GeForce 8800 GTX; later GPUs are expected to reduce time by several orders of magnitude [S1, unofficial].
* **Strengths:** Effective at discriminating structures with similar intensity values but different textures [S1, unofficial].
* **Limitations:** Fixed-size regions cannot distinguish homogeneity vs. inhomogeneity within the region [S1, unofficial].
* **When to use/not:** Use when texture discrimination is needed for intensity-similar structures; avoid when intra-region homogeneity variation is critical [G].

## Background
The method was described in reference [20] of the source material [S1, unofficial]. It builds on the application of pattern recognition concepts (texture) to volumetric visualization [S2, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S2] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
