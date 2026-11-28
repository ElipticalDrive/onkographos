# Rendering Pipelines and Passes

Also called: Render Forward Pass

## Overview
Rendering Pipelines and Passes refers to the structural organization and execution flow of object-order volume rendering within the splatting family, where volumetric data is decomposed into primitives and rasterized to form an image [G]. It solves the problem of efficiently rendering large-scale datasets by leveraging GPU rasterization pipelines and screen-space accumulation to handle transparency and depth occlusion [G].

## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [G].

1.  **Primitive Representation:** The volume is decomposed into primitives [G]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [G].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [G]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [G].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane [G]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
4.  **Compositing:** Contributions are accumulated to form the final image [G]. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [G]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [G].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [G].

## Subtopics

### [Plug-and-Play Ray Marching and Rendering](981-plug-and-play-ray-marching-and-rendering.md)
A modular pipeline interface that decouples acceleration logic from the radiance field representation, allowing efficient, differentiable volumetric rendering via two query functions. Chosen for dynamic scenes requiring ease of extension over raw kernel optimization.

### [Time-Sliced Pipelined Architecture](1435-time-sliced-pipelined-architecture.md)
A hardware design methodology achieving real-time performance by overlapping distinct pipeline stages through time-slicing on shared or replicated functional blocks. Chosen for unified multi-core hardware implementations.

### [Slice-by-Slice (Sheet Buffer) Splatting](2110-slice-by-slice-sheet-buffer-splatting.md)
Organizes splats into cache-sheets aligned parallel to the volume face, compositing slice-by-slice. Chosen to leverage hardware rasterization and avoid per-splat compositing artifacts, though it may introduce popping at steep view angles.

### [Screen Quad Rendering](2435-screen-quad-rendering.md)
The final compositing stage in a two-pass pipeline, where pre-rendered data in off-screen FBOs is composited into the default framebuffer. Chosen to decouple geometry passes from final screen composition and enable post-processing.

### [Rendering Pipeline Architecture](3059-rendering-pipeline-architecture.md)
The structural organization of hardware-accelerated splatting, decomposing data into primitives rasterized to form an image. Chosen for real-time rendering of large-scale volumes using screen-space accumulation.

### [Custom Pass Rendering](3629-custom-pass-rendering.md)
A multi-pass architecture decoupling volumetric traversal and shading from the standard graphics pipeline via specialized shaders. Chosen to separate expensive traversal into dedicated passes for lighting and compositing.

### [Multi-View Splatting Pipeline with Fill-Rate Filtering](4689-multi-view-splatting-pipeline-with-fill-rate-filtering.md)
Generates multi-view 2D images from a 3D point cloud by rasterizing splats and filtering views based on pixel coverage density. Chosen to ensure structured output quality for downstream tasks.

### [DirectXSplat Renderer API](8504-directxsplat-renderer-api.md)
A C++/Direct3D 12 library for rendering 3D Gaussian Splats with explicit resource management. Chosen for high-performance integration into existing D3D12 applications.

### [Ray-Centric and Hardware-Accelerated Rendering](8936-ray-centric-and-hardware-accelerated-rendering.md)
Computes pixel color by tracing rays through the volume and accumulating density/color. Chosen for dense volumes where sorting overheads of splatting are prohibitive.

### [Multi-Pass and Sequential Stage Architectures](10539-multi-pass-and-sequential-stage-architectures.md)
Decomposes rendering into distinct computational phases to manage complexity or memory bandwidth. Chosen when decoupling geometry processing from shading logic or managing complex fragment storage.

### [Interactive Forward Rendering with Approximate Splatting](11491-interactive-forward-rendering-with-approximate-splatting.md)
Provides an efficient approximation of point reprojection using global depth layering and logarithmic blending. Chosen for interactive free-viewpoint rendering with high frame rates.

### [BCC Grid Rendering](4786-bcc-grid-rendering.md)
Renders volumetric data structured on a Body-Centered Cubic grid. Chosen for specific grid topologies requiring optimized sampling.

### [Hierarchical and Multi-Resolution Splatting](5618-hierarchical-and-multi-resolution-splatting.md)
Organizes splats into hierarchical levels of detail. Chosen to optimize rendering performance across varying distances and resolutions.

### [Vertex-Based Leaf Node Derivation](7234-vertex-based-leaf-node-derivation.md)
Derives leaf node data directly from vertex attributes. Chosen for efficient GPU-based data structure generation.

### [Gsplat Transformation](7308-gsplat-transformation.md)
Handles specific transformation pipelines for Gaussian splats. Chosen for optimized coordinate transformations in Gaussian splatting frameworks.

### [Optimization and Culling](8124-optimization-and-culling.md)
Applies frustum and occlusion culling to splat primitives. Chosen to reduce unnecessary rasterization overhead.

### [Hybrid and Parallel Rendering](8125-hybrid-and-parallel-rendering.md)
Combines multiple rendering techniques or parallelizes stages. Chosen for complex scenes requiring mixed rendering strategies.

### [Splat Sorting and Composition](8126-splat-sorting-and-composition.md)
Sorts splats by depth before compositing. Chosen to ensure correct transparency handling in object-order rendering.

### [Silhouette Splatting](8357-silhouette-splatting.md)
Renders only the boundary or silhouette of volumetric primitives. Chosen for stylized rendering or edge detection.

### [Splat Kernels and Footprints](8724-splat-kernels-and-footprints.md)
Defines the mathematical functions and screen-space footprints for splats. Chosen to control visual quality and aliasing characteristics.

### [Significant Voxel Extraction](9025-significant-voxel-extraction.md)
Extracts only relevant voxels for rendering. Chosen to reduce memory bandwidth and computation for sparse volumes.

### [Particle and Fluid Simulation](9118-particle-and-fluid-simulation.md)
Integrates splatting with dynamic particle or fluid simulations. Chosen for real-time visualization of animated volumetric data.

### [Error-aware Streaming Fusion](9936-error-aware-streaming-fusion.md)
Fuses streaming data while tracking rendering error. Chosen for continuous updates in dynamic scenes.

### [Data Structure and Storage](10326-data-structure-and-storage.md)
Manages the memory layout and storage of splat primitives. Chosen to optimize cache coherence and access patterns.

### [AstroSplat Render Modes](12010-astrosplat-render-modes.md)
Provides specialized rendering modes for astronomical data. Chosen for scientific visualization of celestial volumes.

### [Two-Pass Surfel Rendering with Enlarged Splat](12325-two-pass-surfel-rendering-with-enlarged-splat.md)
Uses an enlarged splat in the first pass to capture coverage, composited in a second pass. Chosen to handle overlapping surfels accurately.

### [Stroke-based splatting](12479-stroke-based-splatting.md)
Represents volumes using stroke-like primitives. Chosen for artistic or directional rendering effects.

### [Splat Overlap Detection and Removal](13379-splat-overlap-detection-and-removal.md)
Identifies and resolves overlapping splats. Chosen to prevent visual artifacts from double-counting density.

### [Splat Picking](13399-splat-picking.md)
Implements ray-splat intersection for user interaction. Chosen for interactive selection and manipulation of volumetric elements.

### [Vector Splatting](14545-vector-splatting.md)
Extends splatting to vector fields. Chosen for visualizing flow or directional data.

### [Anti-Aliasing Techniques](16275-anti-aliasing-techniques.md)
Applies filtering and supersampling to splat primitives. Chosen to reduce jagged edges and high-frequency artifacts.

### [Differentiable Rendering](16276-differentiable-rendering.md)
Maintains gradient flow through the splatting pipeline. Chosen for optimization and training of neural radiance fields.

## Background
The concept of splatting as a rendering technique was introduced by Westover in 1989, who mapped view-dependent footprints to a generic footprint table to improve rendering speed [S2, unofficial]. The `msplat_render` function is called to render the scene given camera setup and parameters, returning an output image tensor [S1, unofficial].

## Sources
[S1] Show HN: Msplat – 3D Gaussian Splatting training in ~90s on M4 Max, native Metal, https://github.com/rayanht/msplat, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
