# Image Space Outline Generation

Also called: Edge extraction from depth/normal maps, Hertzmann's outline generation, Outline Post Processing

## Overview
Image Space Outline Generation is a post-processing technique that renders outlines by extracting edges from depth or normal maps, leveraging the observation that depth differences are minimal between adjacent pixels of a single object but large between objects, and normals vary smoothly across a single surface but show discontinuity between surfaces [S1, unofficial]. It differs from analytical methods by operating directly on pixel buffers, making it faster and simpler to implement, though it relies on the depth buffer which may lack the geometric precision of surface-based NPR [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** Depth buffer (Z-buffer), Normal buffer (optional), Edge detection parameters (kernel size, thresholds).
**Outputs:** Binary edge mask or stylized outline texture.

**Data Structures:**
- `DepthBuffer`: 2D array of floating-point depth values ($z$).
- `NormalBuffer`: 2D array of 3D normalized vectors ($\vec{n}$).
- `EdgeMask`: 2D array of boolean or float values indicating edge presence.

**Steps:**
1. **Depth Edge Extraction:**
   - For each pixel $(x, y)$, compute the gradient of the depth buffer using a Sobel filter or similar edge detection kernel [S1, unofficial].
   - Let $D(x, y)$ be the depth value at $(x, y)$.
   - Compute horizontal gradient $G_x$ and vertical gradient $G_y$:
     $$ G_x = D(x+1, y) - D(x-1, y) $$
     $$ G_y = D(x, y+1) - D(x, y-1) $$
   - Compute magnitude $M_d$:
     $$ M_d = \sqrt{G_x^2 + G_y^2} $$
   - If $M_d > T_d$ (threshold), mark $(x, y)$ as a depth edge. Depth edges reflect C0 discontinuities [S1, unofficial].

2. **Normal Edge Extraction (Optional):**
   - For each pixel $(x, y)$, compute the gradient of the normal buffer [S1, unofficial].
   - Let $\vec{N}(x, y)$ be the normal vector at $(x, y)$.
   - Compute horizontal gradient $\vec{G}_x$ and vertical gradient $\vec{G}_y$:
     $$ \vec{G}_x = \vec{N}(x+1, y) - \vec{N}(x-1, y) $$
     $$ \vec{G}_y = \vec{N}(x, y+1) - \vec{N}(x, y-1) $$
   - Compute magnitude $M_n$:
     $$ M_n = \|\vec{G}_x\| + \|\vec{G}_y\| $$
   - If $M_n > T_n$ (threshold), mark $(x, y)$ as a normal edge. Normal edges reflect C1 discontinuities [S1, unofficial].

3. **Combination:**
   - Combine depth and normal edges using logical OR:
     $$ \text{Edge}(x, y) = \text{DepthEdge}(x, y) \lor \text{NormalEdge}(x, y) $$
   - This yields all perceptually significant silhouettes [S1, unofficial].

4. **Post-Processing (Optional):**
   - For stylized lines, calculate analytical curves from the detected edge map, as detected edges only exist as lists of pixels [S1, unofficial].
   - Apply scan conversion algorithms to map parametric curves to screen pixels [S1, unofficial].

**Parameters:**
- $T_d$: Depth threshold. Not given in the sources.
- $T_n$: Normal threshold. Not given in the sources.
- Kernel size: Typically 3x3 for Sobel [S1, unofficial].

**Edge Cases:**
- Precision issues in depth buffer may cause false edges in flat regions.
- Discontinuities at object boundaries may be missed if thresholds are too high.
- [G] Ensure depth buffer is in linear space if non-linear depth values are used.

## Subtopics

### [Close-Up Image Depth Map Generation](13992-close-up-image-depth-map-generation.md)
A method to generate a Close-Up Image Depth Map (CGDM) for images lacking a single vanishing point or containing multiple convergence points, relying on occlusion cues and local edge histograms to infer depth. It differs from perspective view methods by using local edge density rather than global geometric projection, where regions with fewer edges are interpreted as farther away.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of pixels, as it involves a single pass over the depth/normal buffers with constant-time kernel operations.
- **Memory Complexity:** $O(N)$ for storing the edge mask.
- **Performance:** Pixel-based methods are generally faster and simpler to implement than analytical methods [S1, unofficial].
- **Strengths:** Simple implementation, fast execution, works well with depth buffers [S2, unofficial].
- **Limitations:** Relies on depth buffer precision, may produce jagged edges, lacks geometric continuity [S1, unofficial].
- **When to use:** When performance is critical and high-quality analytical edges are not required [S1, unofficial].
- **When not to use:** When precise geometric outlines are needed or when depth buffer is unavailable [S1, unofficial].

## Background
Originating from Hertzmann's survey of image space outline generation algorithms [S1, unofficial], this method has been widely adopted in real-time graphics for its simplicity. It is commonly used in post-processing passes to add outlines to rendered scenes [S2, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] lazergenixdev/First-Voxel-Engine, https://github.com/lazergenixdev/First-Voxel-Engine, fetched 2026-10-26 (unofficial)
