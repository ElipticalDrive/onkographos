# Voxel engines and games
Also called: Voxel frame buffer processors, Voxel engines

## Overview
This family encompasses hardware architectures and software systems designed to manipulate, store, and render volumetric data directly as voxels, rather than converting them to polygons or ray-marching implicit surfaces. They solve the problem of real-time interaction with 3D spatial data by offloading volume-specific operations (clipping, compositing, voxelization) from the main CPU to dedicated processors or optimized GPU pipelines [S1]. Unlike [Film and production rendering](48-film-and-production-rendering.md) which prioritizes offline quality, or [Scientific and medical visualization](49-scientific-and-medical-visualization.md) which prioritizes analytical accuracy, voxel engines prioritize interactive frame rates and direct spatial manipulation of 3D primitives [S1].

## Shared principles
The articles in this family share the goal of mapping abstract volumetric data $I(x,y,z)$ to a perceptible output through specific system constraints. Common mechanisms include:
1.  **Hardware Acceleration Mapping**: Offloading computationally intensive tasks (ray casting, splatting, sorting) to parallel execution units (GPUs, multi-core CPUs) to achieve real-time or near-real-time performance [S1].
2.  **Data Structure Optimization**: Storing volumetric data in formats optimized for the target hardware, such as 3D/4D textures for coalesced memory access on GPUs or sparse structures (e.g., OpenVDB) for memory efficiency [S1].
3.  **Depth Management**: Handling the ordering of contributions for alpha-blended rendering, often requiring GPU-based sorting algorithms (radix sort, linked lists) to ensure correct compositing [S1].
4.  **Physical-to-Digital Mapping**: In the case of volumetric displays, transforming 3D voxel coordinates into 2D control signals (pixel buffers, SLM patterns) synchronized with mechanical or optical processes to create a physical 3D light field [S1].

## Subtopics

### [Voxel Space](3074-voxel-space.md)
A 2.5D rendering technique that generates a pseudo-3D environment by plotting vertical lines whose colors are sampled from a color map and whose starting y-coordinates are determined by a depth map. It distinguishes itself from full 3D rasterization by avoiding polygon projection, instead relying on pre-computed height and color data to simulate perspective, making it suitable for hardware with limited floating-point capabilities.

### [Cascaded Voxel Volumes](4637-cascaded-voxel-volumes.md)
A method for Voxelized Global Illumination (VXGI) that extends the concept of cascaded shadow maps to 3D voxel grids. It uses multiple volumes with identical resolution but different scales, centered around the player, to vary coverage and spatial resolution. Unlike single-volume approaches, it prioritizes high-fidelity voxelization near the eye while maintaining lower-detail coverage for distant indirect illumination. It is distinct from [Voxel Space](3074-voxel-space.md) by explicitly managing multiple discrete resolution tiers rather than a single unified or sparse structure.

## Background
The concept of voxel-based processing originated with the **voxblt Engine (vE)**, a 3D frame-buffer processor developed by **Arie Kaufman** as part of the **CUBE** multiprocessor system in the late 1980s [S1]. The vE was designed as the 3D counterpart to the 2D frame buffer processor (FBP), operating as an independent processor that executes commands from a 3D Frame Buffer Display List (FBDL3) [S1]. It was intended to offload room management, clipping, and primitive operations from the main CPU and 2D video/viewing processors, freeing them for geometry, viewing, and video tasks [S1].

The vE manipulates voxel maps stored in a **Cubic Frame Buffer (CFB)**, a full 3D cellular memory where each voxel holds density, color, material, texture, and translucency data [S1]. A typical CFB for a $512^3$ resolution requires approximately 128 Mbytes of memory [S1]. The system supports three types of primitives: **rooms** (3D windows/cuboids), **jacks** (3D cursors), and **figurines** (3D icons) [S1]. These primitives are unstructured sub-arrays of the CFB, defined by coordinates and priority values to handle space conflicts [S1].

The vE was applied to empirical data in medical imaging, biology, geology, 3D image processing, CAD, scientific volume visualization, and 3D simulation/animation [S1]. It supported 3D input devices such as the Polhemus 3SPACE Isotrak ('kite') and a 2D mouse for 3D positioning [S1]. The architecture included specific tables (Room, Jack, Figurine, Priority, Device) and registers (Mode, System, Input) to manage state, priority, and hardware acceleration [S1].

Modern voxel engines and games have evolved from these hardware-centric origins to software-based implementations on GPUs, utilizing techniques like Cascaded Voxel Volumes for global illumination and 2.5D approximations like Voxel Space for constrained hardware environments.

## Sources
[S1] The voxblt Engine: A Voxel Frame Buffer Processor, https://doi.org/10.2312/eggh/eggh88/085-102, fetched 2026-10-26
