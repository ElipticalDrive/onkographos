# Screen-Space Image Filtering

Also called: Screen-Space Filtering, Screen-Space Techniques

## Overview
Screen-Space Image Filtering modifies a pre-rendered "vacuum" image by applying neighborhood-based filters to simulate effects caused by participating media, using only the pixel data currently available on the screen [S1, unofficial]. Unlike volumetric methods that reconstruct geometry or ray-cast through volume data, this family operates on the final 2D image buffer, treating it as a discrete function to be convolved or adjusted [S1, unofficial]. It is distinct from its siblings because it does not rely on splatting primitives, pre-filtered textures, or specific anti-aliasing kernels during the geometry pass, but rather applies post-processing or screen-space operations to the accumulated color buffer [S1, unofficial].

## Shared principles
The family shares the fundamental mechanism of treating the rendered image as a discrete 2D function and applying spatial filters to it [S1, unofficial].

1.  **Image Representation:** The input is an image $L$, defined as a discrete two-dimensional function $L: \mathbb{N}^2 \rightarrow \mathbb{R}^3$. For each pixel position $\mathbf{x} \in \mathbb{N}^2$, the function returns the color $L(\mathbf{x}) \in \mathbb{R}^3$ [S1, unofficial].
2.  **Color Space and Range:** The color channels (red, green, blue) are assumed to be non-negative real numbers. For High Dynamic Range (HDR) images, the values can range from $0$ to $+\infty$ [S1, unofficial].
3.  **Screen-Space Constraint:** The methods operate exclusively on information currently visible on the screen. They cannot access data for objects behind the camera or occluded geometry that was not rendered in the base pass [S1, unofficial].
4.  **Convolution Basis:** Many techniques in this family apply a filter to the original image, typically using neighborhood-based convolution. The new color of a pixel is determined by its neighbors [S1, unofficial].

## Subtopics

### [Splatting for Anti-aliasing Small Cells](829-splatting-for-anti-aliasing-small-cells.md)
Uses splatting primitives to handle small cells, differing by operating during the geometry pass rather than as a pure screen-space post-process.

### [Motion-Blurred Splats](1343-motion-blurred-splats.md)
Incorporates temporal motion into the splatting process, distinct from static screen-space filtering.

### [Threshold Slicing and Anti-Aliasing](1447-threshold-slicing-and-anti-aliasing.md)
Uses thresholding on volume data for anti-aliasing, a volume-space technique.

### [Prefiltering](2928-prefiltering.md)
Pre-computes filtered textures, differing by moving the filtering cost to a pre-processing stage.

### [PseudoSeparable Cross Bilateral Filter](8374-pseudoseparable-cross-bilateral-filter.md)
Applies a specific bilateral filter variant, a concrete instance of screen-space filtering.

### [Separable Ping-Pong Blur Post-Processing](8804-separable-ping-pong-blur-post-processing.md)
Uses ping-pong buffers for separable blurs, a specific implementation of screen-space convolution.

## Background
The concept of screen-space approaches for rendering participating media was formalized in the work of Elek et al. (2013) and detailed in the thesis "Real-Time Light Transport in Analytically Integrable Participating Media" [S1, unofficial]. These methods became popular in real-time rendering on GPUs because they can be executed in parallel for each output pixel, leveraging the hardware's ability to process image buffers efficiently [S1, unofficial]. The limitation of not accessing off-screen data is often considered acceptable for many real-time applications [S1, unofficial].

## Sources
[S1] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
