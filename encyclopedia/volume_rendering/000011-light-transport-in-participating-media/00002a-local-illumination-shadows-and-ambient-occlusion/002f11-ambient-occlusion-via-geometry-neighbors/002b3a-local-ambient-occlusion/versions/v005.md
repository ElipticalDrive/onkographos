# Local Ambient Occlusion
Also called: Feature-Driven Ambient Occlusion, Pose-Based Ambient Occlusion Approximation

## Overview
Local Ambient Occlusion (LAO) is a family of local illumination techniques that estimate light contributions from neighboring voxels or geometric structures rather than performing global illumination calculations [S1, unofficial][S3]. By restricting occlusion queries to a local vicinity, these methods provide spatial and depth cues that prevent regions from being fully shadowed by distant dense objects (e.g., a skull occluding a tumor) while avoiding the computational cost of global methods [S1, unofficial][S3]. They stand apart from standard local illumination by integrating opacity over the local neighborhood's distribution or sampling directions, and from global methods by ignoring occlusions outside the defined local radius [S1, unofficial][S3].

## Shared principles
The family shares the core mechanism of approximating ambient occlusion by sampling the immediate vicinity of a point $\mathbf{x}$ within the volume or scene, rather than tracing rays to infinity or the entire volume boundary [S1, unofficial][S3].

1.  **Local Neighborhood Definition:** A local region $\Omega$ is defined around the sample point $\mathbf{x}$. This is typically a sphere of radius $R_\Omega$ or a cubic window of size $n \times n \times n$ [S1, unofficial][S3].
2.  **Occlusion Estimation:** The method estimates the amount of light blocked by surrounding geometry. This is done either by:
    *   **Ray Casting:** Casting $K$ rays into the neighborhood and accumulating opacity/translucency along them [S1, unofficial][S3].
    *   **Statistical Integration:** Assuming a probability distribution (e.g., normal) of voxel values within $\Omega$ and integrating the Transfer Function (TF) against this distribution [S3].
3.  **View Independence:** The computed ambient occlusion factor is generally view-independent, allowing it to be precomputed or updated incrementally [S1, unofficial][S3].
4.  **Integration into Shading:** The resulting occlusion factor modulates the final color, typically by multiplying the diffuse or ambient term, or by mixing the base color with a darkened variant [S1, unofficial][S3].

## Subtopics

### [Local Ambient Occlusion for Volume Rendering](4300-local-ambient-occlusion-for-volume-rendering.md)
Computes shadowing by structures in the immediate vicinity of each voxel. It distinguishes itself by using a probabilistic integration approach that assumes CT values follow a normal distribution, accelerated by piecewise linear TF approximation, or a multiresolution ray-casting approach. Choose this for high-speed volume rendering where statistical assumptions hold.

### [Local Ambient Occlusion (LAO)](4502-local-ambient-occlusion-lao.md)
Estimates light contributions by branching rays towards random directions within a hemisphere and tracing voxels to average accumulated translucency. It stands apart by using a two-pass or incremental GPU-based pipeline with specific parameters for step size and light radius. Choose this for real-time GPU implementations requiring soft shadows and emissive tissue support.

### [Feature-Driven Ambient Occlusion](8903-feature-driven-ambient-occlusion.md)
Computes ambient occlusion separately for identified features (layers) to prevent the overdarkening of internal layers caused by inter-occlusions from neighboring features. It sets itself apart by explicitly separating occlusion computation per feature class and using a maximum operator for recombination. Choose this for multi-layer volumes where preserving internal layer visibility is critical.

### Lambertian Lighting with AO and Shadows
Combines Lambertian diffuse shading with Local Ambient Occlusion and Soft Shadows. It computes the normal from the gradient and mixes the base color with darkened variants based on LAO and shadow contributions.

### Deferred Shading for Ambient Occlusion
Renders world position and surface normal pixels to a buffer, then executes AO computation in the fragment shader using deferred shading to avoid evaluating hidden geometry.

### Pose-Based Ambient Occlusion Approximation
Approximates ambient occlusion by modulating Gaussian RGB color based on local body pose, handling self-occlusions and shadows from garment wrinkles. It uses an MLP to output a scaling factor $s \in [0, 2]$ that multiplies the RGB color, clipped to $[0, 1]$.

## Background
The concept of Local Ambient Occlusion in volume rendering originated from the need to enhance depth cues without the prohibitive cost of global illumination. Early implementations focused on statistical approximations of local opacity distributions [S3]. Concurrently, GPU-based techniques emerged to provide real-time LAO for volumetric data, often utilizing multiresolution structures and incremental updates [S1, unofficial][S2]. More recently, the concept has been adapted for neural rendering and Gaussian Splatting, where pose-based approximations handle self-occlusions in animatable avatars [S3].

## Sources
[S1] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S2] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
[S3] Human Gaussian Splatting: Real-Time Rendering of Animatable Avatars, https://doi.org/10.1109/cvpr52733.2024.00081, fetched 2026-10-26
