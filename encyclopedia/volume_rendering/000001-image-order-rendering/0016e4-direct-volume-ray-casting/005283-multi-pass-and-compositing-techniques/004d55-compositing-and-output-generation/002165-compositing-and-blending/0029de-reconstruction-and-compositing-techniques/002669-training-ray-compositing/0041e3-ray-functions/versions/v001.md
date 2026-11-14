# Ray Functions

Also called: Alternative compositing methods, Max/Avg/Distance compositing

## Overview
Ray functions are alternative compositing techniques used in direct volume ray casting to determine the final color along a ray, serving as substitutes for standard alpha compositing [S1, unofficial]. Unlike standard compositing, which integrates opacity and color along the ray depth, ray functions apply a specific aggregation operator (such as maximum, average, or distance-to-value) to the sampled data points to derive the pixel value.

## Shared principles
Ray functions operate on the set of samples $\{s_i\}$ collected along a ray segment intersecting the volume. Each sample $s_i$ typically contains a scalar value (e.g., density or intensity) and potentially color information. The core mechanism replaces the front-to-back or back-to-front alpha blending equation with a direct reduction function $F$ applied to the sample set.

The general form is:
$$ C_{final} = F(\{s_1, s_2, \dots, s_n\}) $$

Where $F$ is chosen from a family of operators. The specific mathematical definitions for the common variants are:

1.  **Maximum**: The final color is determined by the sample with the highest scalar value (often associated with the highest opacity or intensity).
    $$ C_{max} = \max_i (s_i) $$
2.  **Average**: The final color is the arithmetic mean of the sampled values.
    $$ C_{avg} = \frac{1}{n} \sum_{i=1}^{n} s_i $$
3.  **Distance to Value**: This method identifies the first sample along the ray that exceeds a specific threshold value $V_{thresh}$. The color is taken from that sample, and the depth is recorded as the distance to that point.
    $$ \text{if } s_i > V_{thresh} \text{ then } C_{dist} = s_i, \text{ stop} $$

These methods do not inherently account for occlusion or transparency in the same way alpha compositing does; they treat the ray samples as a discrete set of candidates from which a single representative value is selected [S1, unofficial].

## Subtopics

### Maximum Compositing
Selects the sample with the highest intensity or opacity. Useful for highlighting bright structures or surfaces in dense volumes. See [Training Ray Compositing (Sigma)](3868-training-ray-compositing-sigma.md) for related sigma-based approaches.

### Average Compositing
Computes the mean of all samples. Useful for visualizing overall density or intensity distribution without emphasizing outliers.

### Distance to Value
Finds the first sample exceeding a threshold. Useful for surface extraction or isosurface visualization without explicit marching cubes.

## Background
Ray functions were introduced as alternatives to standard compositing techniques in volume rendering pipelines [S1, unofficial]. They are part of the broader category of image-order rendering methods where color is calculated per pixel by casting rays through the 3D data [S1, unofficial].

## Sources
[S1] PPT - Volume Rendering, Part 2 PowerPoint Presentation, free download - ID:2205448, https://slideserve.com/walter/volume-rendering-part-2-powerpoint-ppt-presentation, fetched 2026-10-24 (unofficial)
