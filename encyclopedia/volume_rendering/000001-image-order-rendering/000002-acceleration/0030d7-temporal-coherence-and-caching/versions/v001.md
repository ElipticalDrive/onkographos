# Temporal Coherence and Caching

Also called: Temporal Reuse, Frame-to-Fame Coherence, Dirty Brick Tracking

## Overview
This family accelerates image-order volume rendering by exploiting the similarity between consecutive frames in a sequence, reducing redundant computation by reusing or skipping work that has not changed. It differs from spatial acceleration methods like [Acceleration](18-accelation.md) by operating across time rather than space, and from [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md) by relying on scene stability rather than perceptual error metrics to determine computational effort.

## Shared principles
The core mechanism shared by all children is the identification of "unchanged" or "coherent" state between frame $t$ and frame $t-1$, allowing the renderer to bypass expensive operations (ray marching, shading, or visibility culling) for those regions. This is achieved through one of two primary strategies:
1.  **State Tracking:** Explicitly tracking which parts of the volume data (e.g., bricks, cells) or ray paths have changed since the last frame. If a region is marked "clean" or "unchanged," its previous result is reused or skipped [S1, unofficial][S2, unofficial].
2.  **Temporal Accumulation/Reprojection:** Blending or reprojecting the previous frame's output into the current frame using motion vectors or depth information to stabilize the image and reduce noise or aliasing, effectively amortizing the cost of high-quality rendering over multiple frames [S1, unofficial].

Mathematically, for a pixel $p$, the output $C_t(p)$ is computed as:
$$ C_t(p) = \begin{cases} C_{prev}(p) & \text{if } \text{unchanged}(p) \\ \text{Render}(p) & \text{otherwise} \end{cases} $$
or via accumulation:
$$ C_t(p) = \alpha C_{prev}(p) + (1-\alpha) C_{new}(p) $$
where $\alpha$ is a blending factor based on confidence or motion magnitude [S1, unofficial].

## Subtopics

### [Intra-ray Parallelism](3135-intra-ray-parallelism.md)
A scheme that partitions each ray into depth bins to enable parallel accumulation of primitive contributions. It differs from standard ray marching by decoupling spatial traversal from depth-wise integration, allowing GPU threads to process multiple depth segments of a single ray simultaneously. Choose this when rendering large numbers of primitives (e.g., ellipsoids) where intra-ray parallelism can be exploited on the GPU.

### [Temporal Differential Rendering](5499-temporal-differential-rendering.md)
An acceleration technique that skips rendering screen tiles whose underlying volume data has not changed since the previous frame. It solves the problem of redundant computation in dynamic scenes by tracking "dirty" bricks on the CPU and projecting them into screen tiles. Unlike standard temporal reuse, it ensures correctness by only re-rendering affected regions, skipping both visibility detection and shading passes for clean tiles. Choose this for dynamic volumes with localized changes (e.g., physics simulations).

### [Exploiting Temporal Coherence (C-Buffer)](5560-exploiting-temporal-coherence-c-buffer.md)
A technique for volume animations where the coordinates of the first non-transparent voxel hit by a ray are stored in a C-buffer (Coordinates-buffer) to determine ray start points for subsequent frames. Choose this for scenes with high transparency and coherent camera motion where early termination can be significantly accelerated by reusing previous hit depths.

### [Reprojection with Temporal Accumulation](8337-reprojection-with-temporal-accumulation.md)
A technique to stabilize voxel rendering over time by blending the current frame's color with the accumulated color from the previous frame, using camera motion vectors to align pixels. Choose this for noisy rendering techniques (e.g., sparse sampling) where temporal stability is more important than absolute frame-by-frame accuracy.

### Inter-Ray Coherency
- **Definition:** A reference implementation that tracks rays from the eye point to the first iso-surface intersection or volume exit, using fast voxel traversal and fast ray-surface intersection. It exploits inter-ray coherency using a buffer storing cells encountered along the last ray and a flag indicating if an iso-surface portion is contained there. For rays visiting the same cells, the intersection bit indicates whether an intersection test must be executed or not [S1, unofficial].
- **Performance:** This technique accelerates rendering by about 60 percent [S1, unofficial].

### Image-Based Rendering for Volume Surfaces
- **Definition:** An acceleration technique that caches a fully volume-rendered image (keyview) and reuses it for novel views by texture-mapping it to a surface model, only ray-casting newly visible regions [S2, unofficial].
- **Steps:** 1. Construct an underlying surface model of the volume. 2. Cache the fully volume-rendered image (keyview). 3. Texture map the keyview onto the surface geometry. 4. For a novel view, identify pixels in newly visible regions. 5. Cast rays only for those pixels to update the image [S2, unofficial].
- **Performance:** Provides an average order of magnitude speedup over traditional volume rendering with very little image quality loss; tested on a $512^3$ patient colon dataset for interactive navigation [S2, unofficial].

## Background
The concept of exploiting temporal coherence in volume rendering emerged in the early 2000s as hardware capabilities allowed for more complex dynamic scenes. Neubauer et al. introduced cell-based first-hit ray casting with inter-ray coherency buffers to accelerate iso-surface visualization [S1, unofficial]. Chen proposed image-based rendering for volume surfaces, caching keyviews to accelerate novel view generation in virtual colonoscopy [S2, unofficial]. More recently, techniques like temporal differential rendering have been integrated into real-time engines to handle dynamic volume data (e.g., from physics simulations) by tracking dirty bricks [S1, unofficial].

## Sources
[S1] (PDF) Cell-Based First-Hit Ray Casting - ResearchGate, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S2] Academia | (PDF) FAST VOLUME RENDERING USING A SHEAR-WARP FACTORIZATION ..., https://www.academia.edu/6981092/FAST_VOLUME_RENDERING_USING_A_SHEAR_WARP_FACTORIZATION_OF_THE_VIEWING_TRANSFORMATION, fetched 2026-10-27 (unofficial)
