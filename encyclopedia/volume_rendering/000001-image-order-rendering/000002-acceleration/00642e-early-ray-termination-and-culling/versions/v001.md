# Early Ray Termination and Culling

Also called: Early Ray Termination, ERT, Culling, Space Skipping

## Overview
Early Ray Termination (ERT) and Culling are acceleration techniques that reduce computational cost by halting ray integration when accumulated opacity reaches a threshold (typically 1.0) or by skipping regions of the volume known to be empty or occluded [S1]. Unlike standard ray casting which samples the entire ray length, these methods modify the front-to-back compositing loop to break early, significantly improving performance for opaque or sparse volumes [S1].

## Shared principles
The core mechanism shared by all acceleration methods is the reduction of work units (samples, voxels, or pixels) processed per frame while maintaining acceptable visual fidelity [G]. This is achieved through three primary strategies:
1. **Culling/Skipping:** Identifying regions of the volume or ray path that do not contribute to the final image (e.g., empty space, occluded voxels) and bypassing their evaluation [S2, unofficial].
2. **Termination:** Halting the integration loop when the accumulated contribution becomes negligible (e.g., high opacity, low transmittance) [S1].
3. **Adaptation:** Varying the computational effort based on local scene complexity or global performance budgets [S1].

Mathematically, these methods modify the standard front-to-back compositing loop:
$$ C_{acc} = C_{acc} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
$$ \alpha_{acc} = \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc}) $$
by introducing conditions that break the loop early, skip iterations, or reduce the number of $C_{sample}$ queries.

## Subtopics

### [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)
Uses a 2D opacity map orthogonal to the viewing direction to cull entire sub-volume bounding boxes that are hidden behind opaque surfaces.

### [Inference Ray Marching and Integration](2148-inference-ray-marching-and-integration.md)
A GPU-accelerated method for neural volume rendering that uses occupancy grids for empty-space skipping and adaptive step sizing to maximize inference throughput.

### [SDF Scaling and Early Ray Termination for Fast Sphere Tracing](3807-sdf-scaling-and-early-ray-termination-for-fast-sphere-tracin.md)
Accelerates sphere tracing by using a low-resolution pre-pass with scaled SDFs to provide a "head start" for full-resolution rays, reducing the number of steps required for convergence.

### [Efficient Space Skipping & Adaptive Sampling of Unstructured Volumes Using H.W. Accel. Ray Tracing](8217-efficient-space-skipping-adaptive-sampling-of-unstructured-v.md)
Targets unstructured meshes by leveraging hardware-accelerated ray tracing to skip transparent elements and adaptively sample based on color variance.

### [Early Z-Test Acceleration](8226-early-z-test-acceleration.md)
A hardware optimization that discards occluded fragments before fragment shader execution, integrating ERT and empty-space skipping into the rasterization pipeline.

### [Early Ray Termination with Stereo Coherence](18678-early-ray-termination-with-stereo-coherence.md)
Extends ERT to stereoscopic rendering by bounding the right-eye projection of subsequent left-eye rays to prevent visual errors caused by terminated rays.

### Distribution Model SPH
- [S1] Employs an early z-culling technique to cull empty texels, ensuring no computations are wasted on them.
- [S1] Data dependencies are removed (via Equation 4), allowing density computation to be executed in the same pass as force computation.
- [S1] Uses a distribution model instead of the gathering model used in [TH07], avoiding neighborhood searching via texture lookups.

### Run-Length Encoding for Empty Space Skipping
- [S2, unofficial] A technique that compresses sequences of empty (transparent) voxels into compact representations during ray traversal to allow rays to jump directly to the next non-empty segment, minimizing redundant sampling.
- [S2, unofficial] Scan the volume to encode runs of transparent voxels into compact representations.
- [S2, unofficial] During ray casting, use the encoded runs to jump directly to the next non-empty segment along the ray.
- [S2, unofficial] Achieves speedups of 2 to 5 times on datasets like CT scans when integrated into shear-warp volume rendering.

### Hardware-Based Empty Space Skipping
- [S3, unofficial] An optimization that determines empty space prior to each traversal pass using a CPU-computed 3D texture map encoding emptiness for blocks of voxels, locking fragments in empty regions via z-buffer writes.
- [S3, unofficial] Modify intermediate pass: Before each traversal pass, sample coarse grid at current ray position. Determine visibility by a dependent texture lookup. Lock fragments by writing zfar to z-buffer.
- [S3, unofficial] Data structure: 3D texture map storing minimum and maximum scalar values per block of 3x3x3 cells (8^3 voxels). Parameterized with minimum/maximum values with respect to the current transfer function. Updated on transfer function modification.

## Background
Early ray termination was introduced as a fundamental optimization for volume rendering to handle opaque surfaces efficiently [S1]. The concept of skipping empty space has evolved from simple run-length encoding (RLE) along rays to more complex hierarchical structures like octrees and KD-trees [S2, unofficial]. Hardware-based optimizations, such as early Z-tests, have further integrated these software concepts into the graphics pipeline, allowing for fragment culling before shader execution [S1]. The technique is widely used in medical imaging (CT scans), scientific visualization, and modern neural radiance fields (NeRFs) where inference speed is critical [S2, unofficial][S1].

## Sources
[S1] Adaptive Sampling and Rendering of Fluids on the GPU, https://doi.org/10.2312/vg/vg-pbg08/137-146, fetched 2026-10-26
[S2] volume rendering, https://grokipedia.com/page/Volume_rendering, fetched 2026-10-26 (unofficial)
[S3] Hardware-Based Volume Ray Casting, https://doi.org/10.2312/egt.20031107, fetched 2026-10-26 (unofficial)
