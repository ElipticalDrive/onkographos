# Temporal Coherence and Caching

## Overview
Temporal Coherence and Caching accelerates volume rendering by exploiting the similarity between consecutive frames or adjacent rays, reducing redundant computation through spatial, temporal, or inter-ray state reuse [G]. It sets itself apart from sibling methods like [GL-NeRF](803-gl-nerf.md) and [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md) by prioritizing the reuse of previously computed geometric or color data (via buffers, caches, or reprojection) rather than solely adjusting sampling density or neural inference strategies [G].

## Shared principles
The core mechanism shared by all children is the reduction of work units (samples, voxels, or pixels) processed per frame by leveraging correlation in time or space [G]. This is achieved through three primary strategies:
1. **State Caching:** Storing intermediate results (e.g., traversal states, rendered buffers, octree nodes, or Gaussian parameters) from previous frames or adjacent rays to skip or accelerate current computations [S1, S2, S3, unofficial].
2. **Dirty-Region Tracking:** Identifying and rendering only modified or newly visible regions (e.g., screen tiles, anchor sets) while skipping unchanged areas [S1, unofficial].
3. **Coherence Exploitation:** Assuming that camera motion, volume updates, or ray paths are small between iterations, allowing for early termination, skipped traversal, or reduced refinement [S1, S2, S3, unofficial].

## Subtopics

### [Inter-Ray Coherency](1030-inter-ray-coherency.md)
Exploits spatial correlation between adjacent rays within a single frame to skip redundant traversal and intersection tests.

### [Image-Based Rendering for Volume Surfaces](1383-image-based-rendering-for-volume-surfaces.md)
Caches a keyview and generates novel views by ray-casting only newly visible regions, using geometric surface models for reprojection.

### [Intra-ray Parallelism](3135-intra-ray-parallelism.md)
Partitions each ray into depth bins to enable parallel accumulation of primitive contributions, optimizing within-ray computation.

### [Temporal Differential Rendering](5499-temporal-differential-rendering.md)
Skips computation for screen tiles that have not changed since the previous frame, tracking "dirty" data structures to cull entire tiles.

### [Reprojection with Temporal Accumulation](8337-reprojection-with-temporal-accumulation.md)
Stabilizes rendering by blending current frame color with an accumulated buffer from the previous frame, using motion vectors for alignment.

### [Mega-NeRF-Dynamic Rendering](8907-mega-nerf-dynamic-rendering.md)
Dynamically expands and reuses a sparse voxel octree cache across frames for large-scale NeRF fly-throughs, minimizing latency.

### [Dynamic Cache Depth Scheduling](13915-dynamic-cache-depth-scheduling.md)
Adjusts the depth of reuse for cached computation results based on the intensity of decoding calculations in the current frame to balance performance and visual stability.

### 4D Volume Caching
- **Definition:** A technique for rendering 4D data where all time steps are cached in graphics memory as an array of 3D textures, allowing fast switching between timesteps without data transfer during rendering [S1, unofficial].
- **Steps:**
    1. Allocate an array of `GLuint` textures for `nTimesteps` [S1, unofficial].
    2. For each timestep $i$, bind the texture, set parameters, and upload data using `glTexImage3D` with format `GL_INTENSITY`/`GL_LUMINANCE` and type `GL_UNSIGNED_BYTE` [S1, unofficial].
    3. Initialize `curTimeStep` to 0 [S1, unofficial].
    4. During draw, increment `curTimeStep` modulo `nTimesteps`, bind the corresponding texture from the array, and draw [S1, unofficial].
- **Parameters:** Texture target: `GL_TEXTURE_3D`; Internal format: `GL_INTENSITY` [S1, unofficial].

### Multi-resolution GPU cache for INR
- **Definition:** Addresses the rendering side with a multi-resolution GPU cache for INR-compressed volumes [S2, unofficial].
- **Performance:** Achieves a 5× rendering speedup, sufficient for interactive exploration of a 0.96 TB DNS dataset at 92 FPS on a consumer GPU [S2, unofficial].

### Temporal Coherence Measurement
- **Definition:** Temporal coherence is defined as the mean of individual voxels' standard deviations over a time span $[t_1, t_2]$, treating each voxel as an independent variable [S3, unofficial].
- **Formulas:**
    - Mean: $\text{Mean} = \frac{1}{A} \sum_{t,i,j} v_{ij}^t$, where $v_{ij}^t$ is the value of voxel $i$ at time step $t$, and $A$ is the total number of voxels in the subvolume across all time steps [S3, unofficial].
    - Standard Deviation: $\text{Standard Deviation} = \sqrt{\frac{1}{A} \sum_{t,i,j} (v_{ij}^t - \text{Mean})^2}$, computed in a one-pass bottom-up octree construction manner, equivalent to standard statistical formula [S3, unofficial].
    - Voxel Standard Deviation: $s(v_x) = \sqrt{\frac{1}{t_2 - t_1 + 1} \sum_{t=t_1}^{t_2} (v_x^t - \bar{v}_x)^2}$, where $s(v_x)$ is voxel $v_x$'s standard deviation in time span $[t_1, t_2]$, and $n$ is the number of voxels within the subvolume [S3, unofficial].
    - Temporal Coherence: $\text{Temporal Coherence} = \frac{1}{n} \sum_{x=1}^{n} s(v_x)$, where $s(v_x)$ is the voxel's standard deviation in the time span, and $n$ is the number of voxels in the subvolume [S3, unofficial].
- **Comparison:** This temporal coherence measure is more effective than overall standard deviation in 4D space because spatial domain variation at an individual time step does not affect the measurement, allowing detection of coherence in subvolumes lacking spatial coherence [S3, unofficial].

### Cache-aware Ray Propagation with Thread Block Synchronization
- **Definition:** A method to maximize texture buffer (TB) size to exploit spatial locality, requiring fewer resident TBs. It synchronizes threads of the same thread block (TB) at regular intervals to realize synchronous ray propagation, specifically for viewpoints with relatively large strides [S4, unofficial].
- **Performance:** Experimental results indicate the cache-aware method doubles the worst rendering performance compared to those provided by the CUDA and OpenCL software development kits [S4, unofficial].

### Cache Coherency Reordering
- **Definition:** Reordering tetrahedra when performing the view sort to eliminate cache stalls during rendering, as tetrahedra are accessed by their view and not memory storage order [S5, unofficial].
- **Pitfall:** There was an overall slowdown because the sorting routine was slowed down by moving more data, so future work is needed to find effective caching strategies [S5, unofficial].

## Background
The concept of exploiting temporal coherence in volume rendering originated in the late 1990s and early 2000s, with early works focusing on C-Buffer techniques to accelerate ray casting by reusing hit distances [S3, unofficial]. The introduction of 4D data rendering led to caching strategies that store entire time-series in GPU memory to avoid PCIe bottlenecks [S1, unofficial]. More recently, with the rise of Neural Radiance Fields (NeRFs) and Implicit Neural Representations (INRs), temporal coherence has been adapted to dynamic octree expansion and multi-resolution GPU caching to handle large-scale interactive visualization [S2, S1, unofficial].

## Sources
[S1] PDF Slide 1, https://www.nvidia.com/content/GTC/documents/1102_GTC09.pdf, fetched 2026-10-29 (unofficial)
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S3] NASA Technical Reports Server (NTRS) 20000069008: Fast Time-Varying Volume Rendering Using Time-Space Partition (TSP) Tree, https://archive.org/details/NASA_NTRS_Archive_20000069008, fetched 2026-10-26 (unofficial)
[S4] PDF Volume Rendering Methods for Computational Fluid Dynamics Visualization, https://www.researchgate.net/profile/David-Ebert-4/publication/220943451_Volume_Rendering_Methods_for_Computational_Fluid_Dynamics_Visualization/links/00b7d51a73e2892e23000000/Volume-Rendering-Methods-for-Computational-Fluid-Dynamics-Visualization.pdf, fetched 2026-10-27 (unofficial)
[S5] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
