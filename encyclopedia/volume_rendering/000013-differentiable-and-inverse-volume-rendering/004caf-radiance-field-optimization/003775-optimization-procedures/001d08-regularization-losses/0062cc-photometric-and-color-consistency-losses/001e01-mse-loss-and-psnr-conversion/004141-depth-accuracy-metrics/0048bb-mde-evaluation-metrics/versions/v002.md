# MDE evaluation metrics

Also called: Depth consistency metrics, Monocular Depth Estimation metrics

## Overview
A set of quantitative metrics used to evaluate the accuracy of predicted depth maps against ground truth, plus a consistency metric for multi-view alignment [S1, unofficial]. These metrics assess the fidelity of monocular depth estimation modules within radiance field optimization pipelines, distinguishing between per-image accuracy and global scene consistency [S1, unofficial].

## Shared principles
The evaluation framework relies on comparing a predicted depth map $\bar{D}$ against a ground truth depth map $D^{GT}$ at each pixel coordinate $p$ [S1, unofficial]. The metrics are computed over the set of valid pixels in an image $I$ [S1, unofficial]. For multi-view consistency, a scene-level scaling factor $s$ is computed to align predicted depths with ground truth without per-frame fitting [S1, unofficial]. This factor is derived from the median ratio of ground truth to predicted depth across all images in a scene [S1, unofficial]. Let $S$ be the set of images belonging to a single scene, and $N = |S|$ be the total number of images in $S$ [S1, unofficial]. The scaling factor $s$ is defined as:

$$
s = \frac{1}{N} \sum_{I_i \in S} \text{median}\left(\frac{D_i^{GT}}{\bar{D}_i}\right)
$$

where $D_i^{GT}$ and $\bar{D}_i$ are the ground truth and predicted depth maps for image $I_i$ [S1, unofficial]. This metric evaluates view consistency by applying a single scale factor to the entire scene rather than optimizing per-frame [S1, unofficial].

## Subtopics

### AbsRel (Mean Absolute Relative Error)
Measures the mean absolute relative difference between predicted and ground truth depth [S1, unofficial].
$$
\text{AbsRel} = \frac{1}{|I|} \sum_{p \in I} \frac{|\bar{D}(p) - D^{GT}(p)|}{D^{GT}(p)}
$$

### SqRel (Mean Squared Relative Error)
Computes the mean of the squared relative differences [S1, unofficial].
$$
\text{SqRel} = \frac{1}{|I|} \sum_{p \in I} \frac{|\bar{D}(p) - D^{GT}(p)|^2}{D^{GT}(p)}
$$

### RMSE (Root Mean Squared Error)
The square root of the mean squared error between predicted and ground truth depths [S1, unofficial].
$$
\text{RMSE} = \sqrt{\frac{1}{|I|} \sum_{p \in I} |\bar{D}(p) - D^{GT}(p)|^2}
$$

### RMSE log (Logarithmic RMSE)
Computes the RMSE on the logarithmic scale of depths [S1, unofficial].
$$
\text{RMSE log} = \sqrt{\frac{1}{|I|} \sum_{p \in I} |\log \bar{D}(p) - \log D^{GT}(p)|^2}
$$

## Background
These metrics are used to evaluate monocular depth estimation modules within radiance field optimization pipelines [S1, unofficial]. The specific formulation for the scene-level consistency scaling factor $s$ follows prior work cited as [64] in the source [S1, unofficial]. The implementation details, such as the use of the median for consistency evaluation, were adopted to ensure fair comparison with relevant prior work [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
