# MDE evaluation metrics

Also called: Depth consistency metrics, Monocular Depth Estimation metrics

## Overview
A set of quantitative metrics used to evaluate the accuracy of predicted depth maps against ground truth, plus a consistency metric for multi-view alignment [S1, unofficial]. These metrics assess the fidelity of monocular depth estimation (MDE) modules within radiance field optimization pipelines, distinguishing between per-image accuracy and global scene consistency [S1, unofficial].

## Shared principles
The evaluation framework relies on comparing a predicted depth map $\bar{D}$ against a ground truth depth map $D^{GT}$ at each pixel coordinate $p$ [S1, unofficial]. The metrics are computed over the set of valid pixels in an image $I$ [S1, unofficial].

For multi-view consistency, a scene-level scaling factor $s$ is computed to align predicted depths with ground truth without per-frame fitting [S1, unofficial]. This factor is derived from the median ratio of ground truth to predicted depth across all images in a scene [S1, unofficial].

Let $S$ be the set of images belonging to a single scene, and $N = |S|$ be the total number of images in $S$ [S1, unofficial]. The scaling factor $s$ is defined as:

$$
s = \frac{1}{N} \sum_{I_i \in S} \text{median}\left(\frac{D_i^{GT}}{\bar{D}_i}\right)
$$

where $D_i^{GT}$ and $\bar{D}_i$ are the ground truth and predicted depth maps for image $I_i$ [S1, unofficial]. This metric evaluates view consistency by applying a single scale factor to the entire scene rather than optimizing per-frame [S1, unofficial].

## Subtopics

### AbsRel (Mean Absolute Relative Error)
Measures the mean absolute relative difference between predicted and ground truth depth. It is sensitive to depth variations in near-field objects [S1, unofficial].

### SqRel (Mean Squared Relative Error)
Computes the mean of the squared relative differences. It penalizes large errors more heavily than AbsRel [S1, unofficial].

### RMSE (Root Mean Squared Error)
The square root of the mean squared error between predicted and ground truth depths. It provides a global error magnitude in depth units [S1, unofficial].

### RMSE log (Logarithmic RMSE)
Computes the RMSE on the logarithmic scale of depths. It is less sensitive to large depth values and focuses on relative accuracy in log space [S1, unofficial].

## Background
These metrics were formalized in the context of boosting radiance fields from sparse inputs using monocular depth adaptation [S1, unofficial]. The specific formulation for the scene-level consistency scaling factor $s$ follows prior work cited as [64] in the source [S1, unofficial]. The implementation details, such as the use of the median for consistency evaluation, were adopted to ensure fair comparison with relevant prior work [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
