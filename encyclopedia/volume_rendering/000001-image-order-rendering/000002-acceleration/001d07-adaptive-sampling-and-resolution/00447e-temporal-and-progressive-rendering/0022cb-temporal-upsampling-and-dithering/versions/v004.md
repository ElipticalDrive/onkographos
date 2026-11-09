# Temporal Upsampling and Dithering

Also called: Temporal Upsampling

## Overview
Temporal Upsampling and Dithering is a rendering technique that applies blue noise dithering to solve slicing artifacts caused by insufficient sampling steps in volume rendering [S1, unofficial]. It combines this dithering with temporal upsampling to reduce the noise introduced by the dithering process [S1, unofficial]. It distinguishes itself from static low-resolution rendering by leveraging temporal accumulation to reconstruct high-resolution details, and from pure dithering by mitigating visual noise through frame-to-frame blending [S1, unofficial].

## How it works
**Inputs and Outputs:** Not given in the sources. [S1, unofficial]
**Data Structures:** Not given in the sources. [S1, unofficial]
**Steps:**
1. Render the volume at a reduced resolution (e.g., $1/2 \times 1/2$) to lower computational cost [S1, unofficial].
2. Apply blue noise dithering to the low-resolution samples to break up coherent aliasing patterns and slicing artifacts caused by insufficient sampling steps [S1, unofficial].
3. Combine the dithered result with temporal upsampling to reduce the noise introduced by the dithering process [S1, unofficial].
**Formulas:** Not given in the sources. [S1, unofficial]
**Parameters:** Not given in the sources. [S1, unofficial]
**Edge Cases and Pitfalls:** Not given in the sources. [S1, unofficial]

## Subtopics

### [Temporal Anti-Aliasing with Anti-Smear](3614-temporal-anti-aliasing-with-anti-smear.md)
Temporal Anti-Aliasing with Anti-Smear is a post-processing technique that blends the current frame's color with an accumulated history buffer using pixel velocity for reprojection, with an optional anti-smear feature that discards accumulated pixels with large depth differences to prevent ghosting. It distinguishes itself from standard TAA by explicitly checking depth discontinuities during reprojection to avoid smearing artifacts.

### Jittered Dithering Distributions
Jittered Dithering Distributions is an algorithm for quickly generating a pseudo-uniform distribution of points based on a regular 2D grid, originally proposed by Mitchell [S2, unofficial]. Create a regular 2D grid with sixteen times as many grid cells as the desired number of points [S2, unofficial]. Visit each cell in turn in a left-to-right and top-to-bottom fashion [S2, unofficial]. Calculate potential $D_{i,j}$ using the function $D_{i,j} = (4D_{i-1,j} + D_{i-1,j-1} + 2D_{i,j-1} + D_{i+1,j-1}) / 8 + R$, where $R = \text{random}()$ [S2, unofficial]. If $D_{i,j}$ is greater than the threshold of one-half, select a jittered point at cell $(i,j)$ and reduce the $D_{i,j}$ potential by one, thus selecting approximately 1/16th of the data points [S2, unofficial]. Modifying the threshold or random increment to generate weighted point densities produces regular patterns in the splats and causes the number of dots deposited to vary widely from the expected number [S2, unofficial].

### Spatial Dithering for Color Perception
Spatial Dithering for Color Perception is a method using spatial dithering to create the perception of more than 3-bit color from single-bit-depth SLM frames [S3]. The system uses a 3-SLM projection engine where each SLM has 1-bit depth [S3]. Proprietary rasterization and half-toning/dithering methods process the OpenGL pipeline data before projection [S3].

## Operating characteristics
* **Time Complexity:** Not given in the sources. [S1, unofficial]
* **Memory Complexity:** Not given in the sources. [S1, unofficial]
* **Strengths:** Not given in the sources. [S1, unofficial]
* **Limitations:** Not given in the sources. [S1, unofficial]

## Background
This method is used in real-time volume rendering, particularly for atmospheric effects like clouds, where performance is critical [S1, unofficial]. It builds on the principles of adaptive sampling and resolution [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md).

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] Multivariate volume rendering, https://www.osti.gov/biblio/495676, fetched 2026-10-26 (unofficial)
[S3] 100-million-voxel volumetric display, https://doi.org/10.1117/12.480930, fetched 2026-11-01
