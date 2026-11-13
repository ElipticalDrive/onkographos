# Screen-Space Image Filtering

Also called: Screen-space approaches

## Overview
Screen-Space Image Filtering refers to a family of techniques that modify a base image rendered in a vacuum by adding effects caused by participating media, using only information currently visible on the screen [S1, unofficial]. What sets this family apart from its siblings is its exclusive reliance on the 2D image buffer; it cannot access data for objects behind the camera or occluded geometry that was not rendered in the base pass [S1, unofficial]. This constraint allows for high parallelism on GPUs but limits physical accuracy compared to volume-order methods.

## Shared principles
The family shares the fundamental mechanism of treating the rendered image or volumetric data as a discrete function that requires smoothing to approximate the continuous signal [S1, unofficial].

1.  **Image Representation**: The input is an image $L$, defined as a discrete two-dimensional function $L: \mathbb{N}^2 \rightarrow \mathbb{R}^3$. For each pixel position $\mathbf{x} \in \mathbb{N}^2$, the function returns the color $L(\mathbf{x}) \in \mathbb{R}^3$ [S1, unofficial].
2.  **Color Space and Range**: The color channels (red, green, blue) are assumed to be non-negative real numbers. For High Dynamic Range (HDR) images, the values can range from $0$ to $+\infty$ [S1, unofficial].
3.  **Screen-Space Constraint**: Many techniques in this family operate exclusively on information currently visible on the screen. They cannot access data for objects behind the camera or occluded geometry that was not rendered in the base pass [S1, unofficial].
4.  **Convolution Basis**: Many techniques apply a filter to the original image or splat footprint, typically using neighborhood-based convolution. The new color or contribution of a pixel is determined by its neighbors [S1, unofficial].

## Subtopics

### [Splatting for Anti-aliasing Small Cells](829-splatting-for-anti-aliasing-small-cells.md)
Uses screen-space splatting to handle small cells.

### [Motion-Blurred Splats](1343-motion-blurred-splats.md)
Incorporates motion blur into the screen-space splatting process.

### [Threshold Slicing and Anti-Aliasing](1447-threshold-slicing-and-anti-aliasing.md)
Applies thresholding to slice data before screen-space filtering.

### [Prefiltering](2928-prefiltering.md)
Pre-computes filters to accelerate screen-space operations.

### [PseudoSeparable Cross Bilateral Filter](8374-pseudoseparable-cross-bilateral-filter.md)
Uses a pseudo-separable bilateral filter for edge-preserving smoothing.

### [Separable Ping-Pong Blur Post-Processing](8804-separable-ping-pong-blur-post-processing.md)
Implements blur via separable ping-pong passes.

## Background
The term "screen-space" was introduced to describe methods that modify an image rendered in a vacuum by adding effects caused by participating media, relying solely on information "on the screen" [S1, unofficial]. These approaches became popular in real-time rendering on GPUs because they can often be executed for each output pixel individually in parallel [S1, unofficial]. The limitation of not working with information not currently on the screen (e.g., objects behind the camera) is noted, though often considered not critical in certain conditions [S1, unofficial]. The mathematical foundation treats the image as a discrete 2D function $L: \mathbb{N}^2 \rightarrow \mathbb{R}^3$ with HDR values in $[0, +\infty)$ [S1, unofficial].

## Sources
[S1] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
