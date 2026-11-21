# Binary and Soft Classification

Also called: Binary Classifier, Non-Binary Classifier, Soft Classification

## Overview
Binary and Soft Classification evaluates input signals to determine voxel inclusion, producing either sharp transitions or smooth gradients in the output [S1, unofficial]. Binary classification makes hard "yes" or "no" decisions based on thresholds, while soft (non-binary) classification uses smooth transitions to avoid high-frequency artifacts [S1, unofficial]. This family focuses on the binary or continuous decision boundary for scalar values, differing from [Style Transfer Function (1D)](777-style-transfer-function-1d.md) which emphasizes stylistic mapping, and from [Boundary-Enhanced Opacity](795-boundary-enhanced-opacity.md) which operates on gradient-derived features.

## Shared principles
The core mechanism involves evaluating a scalar input value $v$ against a defined criterion to produce an output value $y$ [G].

1. **Input Domain**: The input is a signal $f_{\text{input}}(\mathbf{x})$ representing the data value at position $\mathbf{x}$ [S1, unofficial].
2. **Decision Boundary**: A threshold or range $[t_{\min}, t_{\max}]$ defines the region of interest [Binary Classification](931-binary-classification.md).
3. **Output Range**: The output is typically a value in $[0, 1]$ representing inclusion probability or opacity [G].
4. **Spectral Impact**: The shape of the classification function directly affects the frequency content of the rendered image [S1, unofficial].

## Subtopics

### [Piecewise-Linear Classification](881-piecewise-linear-classification.md)
Generalizes segmentation by applying per-channel piecewise linear functions to map input tensor values to RGBA formats, allowing continuous control over opacity and color rather than binary inclusion. Choose this when fine-grained, multi-channel control over the transfer function is required.

### [Binary Classification](931-binary-classification.md)
Categorizes voxels into two distinct groups (0-voxels and 1-voxels) based on a threshold, discarding 0-voxels to reduce memory consumption and accelerate rendering. Choose this for real-time rendering of large volumes where performance is critical and sharp boundaries are acceptable.

### Non-Binary Classifier
A classifier that smoothly transitions from "no" to "yes" with varying degrees of acceptance, generating soft edges rather than sharp transitions [S1, unofficial]. It avoids sharp corners in the classifier function, which significantly reduces high frequencies in the output spectrum and minimizes aliasing artifacts compared to binary classifiers [S1, unofficial].

### [Multiple Material Classification](6343-multiple-material-classification.md)
A technique to classify more than one material in a volume by using multiple classifiers, allowing different materials to have different opacity values (e.g., high opacity for an organ, low for context) [G].

## Background
Binary classification was introduced in the context of parallel, feed-forward volume rendering algorithms [S1, unofficial]. The concept of non-binary (soft) classification was developed to mitigate the aliasing artifacts caused by the sharp transitions inherent in binary methods [S1, unofficial]. Multiple material classification extends this approach to handle distinct anatomical structures or materials within the same volume [G].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S2] An Intelligent System Approach to Higher-Dimensional Classification of Volume Data, https://doi.org/10.1109/tvcg.2005.38, fetched 2026-10-27
