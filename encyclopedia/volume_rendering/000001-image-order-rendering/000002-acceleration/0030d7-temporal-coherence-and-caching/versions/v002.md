# Temporal Coherence and Caching

Also called: Temporal Reuse, Frame-to-Fame Optimization

## Overview
Temporal Coherence and Caching accelerates volume rendering in dynamic scenes or animations by exploiting the similarity between consecutive frames, reducing redundant computation by reusing data from previous frames [S1, unofficial][S2, unofficial]. Unlike spatial acceleration methods that optimize the traversal within a single frame, this family focuses on amortizing costs over time, setting it apart from siblings like [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md) which vary effort per frame based on complexity, and [Early Z-Test Acceleration](8226-early-z-test-acceleration.md) which relies on depth buffer occlusion within a single pass.

## Shared principles
The core mechanism shared by all children is the storage of rendering state (geometric hits, color buffers, or traversal caches) from frame $t-1$ and its application to frame $t$ to skip or accelerate ray marching. This relies on the assumption that camera motion and volume changes are small between frames. Mathematically, if $I_{t-1}$ is the previous image and $M$ is the motion field or transformation, the current image $I_t$ is approximated by:
$$ I_t \approx \text{Blend}(I_{t-1} \circ M, I_{\text{current}}) $$
or by starting ray traversal at a depth $d_t$ derived from $d_{t-1}$:
$$ d_t = d_{t-1} + \Delta d $$
where $\Delta d$ accounts for camera movement.

## Subtopics

### [Inter-Ray Coherency](1030-inter-ray-coherency.md)
Exploits spatial similarity between adjacent rays within a single frame by caching traversal states.

### [Image-Based Rendering for Volume Surfaces](1383-image-based-rendering-for-volume-surfaces.md)
Caches a full keyview and projects it onto a surface model for novel views.

### [Intra-ray Parallelism](3135-intra-ray-parallelism.md)
Partitions rays into depth bins for parallel accumulation, building on spatial partitioning logic.

### [Temporal Differential Rendering](5499-temporal-differential-rendering.md)
Skips computation for screen tiles that have not changed since the previous frame.

### [Exploiting Temporal Coherence (C-Buffer)](5560-exploiting-temporal-coherence-c-buffer.md)
Stores 3D hit coordinates to start ray marching deeper in subsequent frames.

### [Reprojection with Temporal Accumulation](8337-reprojection-with-temporal-accumulation.md)
Blends current frame color with previous frame using motion vectors to stabilize noise.

### [Mega-NeRF-Dynamic Rendering](8907-mega-nerf-dynamic-rendering.md)
Dynamically expands a sparse voxel octree cache across frames for NeRFs.

### 4D Volume Caching
- **Definition:** A technique for rendering 4D data where all time steps are cached in graphics memory as an array of 3D textures, allowing fast switching between timesteps without data transfer during rendering [S1, unofficial].
- **Steps:** Allocate an array of `GLuint` textures for `nTimesteps`. For each timestep $i$, bind the texture, set parameters, and upload data using `glTexImage3D` with format `GL_INTENSITY`/`GL_LUMINANCE` and type `GL_UNSIGNED_BYTE`. Initialize `curTimeStep` to 0. During draw, increment `curTimeStep` modulo `nTimesteps`, bind the corresponding texture from the array, and draw [S1, unofficial].
- **Parameters:** Texture target: `GL_TEXTURE_3D`. Internal format: `GL_INTENSITY` [S1, unofficial].

### Multi-resolution GPU cache for INR
- **Definition:** Addresses the rendering side with a multi-resolution GPU cache for INR-compressed volumes [S2, unofficial].
- **Performance:** Achieves a 5× rendering speedup, sufficient for interactive exploration of a 0.96 TB DNS dataset at 92 FPS on a consumer GPU [S2, unofficial].

## Background
The concept of exploiting temporal coherence in volume rendering dates back to the early 2000s, with foundational work on C-Buffer techniques for interactive visualization [S1, unofficial][S2, unofficial]. Recent developments have extended these principles to Neural Radiance Fields (NeRFs) and large-scale scientific data, such as Direct Numerical Simulation (DNS) datasets, leveraging modern GPU memory bandwidth and caching mechanisms [S1, unofficial][S2, unofficial].

## Sources
[S1] PDF Slide 1, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
