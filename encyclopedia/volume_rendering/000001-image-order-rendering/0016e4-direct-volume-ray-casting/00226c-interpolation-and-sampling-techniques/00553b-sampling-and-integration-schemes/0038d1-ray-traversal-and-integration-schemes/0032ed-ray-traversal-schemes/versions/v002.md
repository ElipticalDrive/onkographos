# Ray Traversal Schemes

Also called: Accumulate Projection, Depth Projection, Max Intensity Projection, Ray Traversal -Accumulate

## Overview
Ray traversal schemes aggregate scalar or color values along a ray to produce a 2D image, serving as the final compositing step in direct volume ray casting. Unlike [Ray Marching and Discretization](2788-ray-marching-and-discretization.md) which focuses on sample placement, or [LOS-integration for visualization](4807-los-integration-for-visualization.md) which handles physical light transport, these schemes define the mathematical aggregation function (e.g., max, average, or alpha compositing) applied to the discrete samples. They are distinguished by their specific aggregation logic, ranging from simple extrema extraction to physically-based opacity accumulation.

## Shared principles
All ray traversal schemes operate on a sequence of samples $s_i$ collected along a ray. Each sample $i$ provides an intensity value $I_i$ (or color) and an opacity value $\alpha_i$ derived from transfer functions. The schemes share the following common structure:

1.  **Input**: A sequence of samples $(I_i, \alpha_i)$ for $i = 0 \dots N-1$.
2.  **Order**: Samples are processed in a specific order (typically front-to-back for accumulation, or any order for extrema).
3.  **Output**: A single pixel value $(C, A)$ or scalar value.

The general compositing equation for front-to-back rendering, which underpins the Accumulate scheme, is:
$$
C = C_s \cdot \alpha_s \cdot (1 - A) + C
$$
$$
A = \alpha_s \cdot (1 - A) + A
$$
where $C_s, \alpha_s$ are the sample color and opacity, and $C, A$ are the accumulated color and opacity [S2, S3, unofficial].

## Subtopics

### Intensity
Extracts the raw scalar value of the sample. Used as a base for other projections.

### Max (Maximum Intensity Projection)
Projects the maximum intensity value along the ray. It is used for Magnetic Resonance Angiogram (MRA) to highlight high-intensity structures like blood vessels [S2, S3, unofficial].

### Average
Produces an X-ray picture by averaging values along the ray. It provides a global view of density but loses depth information [S1, S2, S3, unofficial].

### Accumulate
Accumulates opacity while compositing colors to make transparent layers visible. This is the standard volumetric compositing method, allowing visibility of internal structures through semi-transparent volumes. Originated with Levoy '88 [S2, S3, unofficial].

### First
Extracts the first sample that exceeds a threshold or is non-empty. Used to extract iso-surfaces or the nearest surface [S1, unofficial].

### Depth
Records the distance or index of the sample where the ray terminates or a condition is met. Used for depth buffering or surface localization [S1, S2, S3, unofficial].

## Background
Ray traversal schemes are fundamental to Direct Volume Rendering (DVR). The Accumulate scheme was formalized by Levoy in 1988, establishing the standard front-to-back alpha compositing approach [S2, S3, unofficial]. Max Intensity Projection (MIP) and Average projections are older techniques adapted from projectional radiology and CT visualization. The "First" and "Depth" schemes are often used for auxiliary tasks like surface extraction or depth cueing. These schemes are implemented in visualization toolkits like VTK, where they form the core of the ray casting pipeline [S2, S3, unofficial].

## Sources
[S1] PPT - Direct Volume Rendering (DVR): Ray-casting PowerPoint Presentation - ID:465503, https://slideserve.com/jariath/direct-volume-rendering-dvr-ray-casting, fetched 2026-10-24 (unofficial)
[S2] Direct Volume Rendering Techniques, https://www.scribd.com/presentation/380929057/10-raycasting, fetched 2026-10-25 (unofficial)
[S3] Direct Volume Rendering (DVR): Ray-casting | PPT - SlideShare, https://www.slideshare.net/slideshow/direct-volume-rendering-dvr-raycasting/265104120, fetched 2026-10-26 (unofficial)
