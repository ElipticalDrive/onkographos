# Screen-Space Image Filtering

Also called: Screen-space approaches

## Overview
Screen-Space Image Filtering refers to methods that modify an original image rendered in a vacuum by adding effects caused by participating media, using only information currently available on the screen [S1, unofficial]. This approach is distinct from volume-integration or ray-marching techniques because it operates on the 2D rasterized output rather than the 3D scene geometry or density fields. It is primarily chosen for real-time rendering on GPUs due to its parallelizability, though it is limited by the lack of information for objects behind the camera or occluded from the current view [S1, unofficial].

## Shared principles
The family shares a common mathematical model and operational constraint.

**Operational Constraint:**
The method assumes a base image $I_{base}$ has already been rendered using a standard rendering approach (e.g., rasterization of opaque surfaces) [S1, unofficial]. The screen-space filter $F$ is applied to this image to produce the final image $I_{final}$:
$$ I_{final}(x) = F(I_{base})(x) $$
where $x$ is the pixel coordinate [S1, unofficial]. The filter $F$ cannot access scene data (depth buffers, normals, or volume density) that is not implicitly encoded in the color values of $I_{base}$ or explicitly provided as auxiliary buffers, but strictly speaking, the definition emphasizes using "information that are 'on the screen'" [S1, unofficial].

**Mathematical Model:**
An image is treated as a discrete two-dimensional function $L: \mathbb{N}^2 \rightarrow \mathbb{R}^3$ [S1, unofficial].
- For each pixel position $x \in \mathbb{N}^2$, the function returns the color $L(x) \in \mathbb{R}^3$.
- The color components (Red, Green, Blue) are non-negative real numbers.
- The domain is the set of integer pixel coordinates $\mathbb{N}^2$.

**Dynamic Range:**
The method supports High Dynamic Range (HDR) images [S1, unofficial].
- HDR is defined as images where the ratio between the darkest and lightest pixel's color is very high.
- Consequently, each color channel is assumed to have values in the range $[0, +\infty)$ [S1, unofficial].

**Implementation Domain:**
The operations are typically executed for each output pixel individually in parallel, leveraging GPU architectures [S1, unofficial].

## Subtopics

### [Splatting for Anti-aliasing Small Cells](829-splatting-for-anti-aliasing-small-cells.md)
Uses splatting techniques to handle aliasing at small cell sizes within the screen-space framework.

### [Motion-Blurred Splats](1343-motion-blurred-splats.md)
Incorporates motion blur effects into the splatting process for temporal anti-aliasing.

### [Threshold Slicing and Anti-Aliasing](1447-threshold-slicing-and-anti-aliasing.md)
Applies thresholding and slicing techniques to manage aliasing artifacts.

### [Prefiltering](2928-prefiltering.md)
Pre-filters the input data before screen-space processing to reduce high-frequency noise.

### [PseudoSeparable Cross Bilateral Filter](8374-pseudoseparable-cross-bilateral-filter.md)
Uses a pseudo-separable bilateral filter approach for edge-preserving smoothing.

### [Separable Ping-Pong Blur Post-Processing](8804-separable-ping-pong-blur-post-processing.md)
Implements a separable blur using ping-pong buffer techniques for efficient post-processing.

## Background
The concept of screen-space approaches originated in the context of real-time light transport in analytically integrable participating media [S1, unofficial]. The terminology "screen-space" was adopted to distinguish these methods from volume-based techniques, highlighting their reliance on the 2D image plane [S1, unofficial]. These methods gained popularity in real-time rendering on GPUs, as noted by Elek et al. (2013) [S1, unofficial]. The primary motivation was to enable parallel execution per pixel, making complex light transport effects feasible in real-time applications despite the limitation of not having access to occluded scene data [S1, unofficial].

## Sources
[S1] Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
