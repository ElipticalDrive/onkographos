# Hybrid and multi-mode rendering

Also called: Hybrid rendering, Multi-mode rendering

## Overview
Hybrid and multi-mode rendering addresses the limitations of pure volume or pure surface rendering by combining them to leverage the strengths of each. It solves the problem of rendering complex scenes containing both volumetric data and explicit surface geometry, or scenes requiring both sharp foreground surfaces and transparent background structures, by integrating ray-triangle intersections, implicit surface tracing, and volume integration [S1, 2165-multi-mode-rendering-with-iso-surfaces-and-meshes, 8367-hybrid-isosurface-dvr-rendering]. It sets itself apart from pure volume rendering by using surface geometry to clip or guide volume integration, and from pure surface rendering by retaining the ability to render internal volume structures or neural radiance fields behind or within the surface boundary [2165-multi-mode-rendering-with-iso-surfaces-and-meshes, 8367-hybrid-isosurface-dvr-rendering, 11555-mesh-based-neural-rendering].

## Shared principles
All methods in this family share the fundamental mechanism of image-order rendering: for each pixel, a ray is defined in world space, intersected with the volume domain, and sampled along its path to detect the first intersection with the isosurface [G]. The ray is parameterized as $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the direction [G]. The traversal order is determined by the pixel's position in the image plane [G].

The core detection logic relies on the isosurface condition. For a scalar field $f(\mathbf{p})$, the surface is defined by $f(\mathbf{p}) = \tau$. In discrete voxel data, this is typically detected by a sign change or threshold crossing between consecutive samples: $f(\mathbf{p}_1) \cdot f(\mathbf{p}_2) < 0$ relative to $\tau$, or $f(\mathbf{p}) \ge \tau$ [S3, unofficial][S1]. Upon detection, the intersection position is often refined (e.g., via linear interpolation or binary search) to reduce discretization artifacts [S3, unofficial][S6, unofficial]. The surface normal is computed from the gradient $\nabla f$ at the hit point, usually via finite differences or tri-cubic filtering, to enable Phong-like shading [S3, unofficial][S1]. The final color is computed using local lighting models (ambient, diffuse, specular) and potentially ambient occlusion or shadows, with no contribution from voxels behind the surface [S3, unofficial][S5, unofficial].

## Subtopics

### [Multi-Mode Rendering with Iso-surfaces and Meshes](2165-multi-mode-rendering-with-iso-surfaces-and-meshes.md)
This method renders scenes containing both volumetric data and explicit surface geometry (triangle meshes or implicit iso-surfaces) by combining hardware-accelerated ray-triangle intersection with volume ray marching. It solves the problem of correctly occluding volume samples with surface geometry, ensuring that rays do not integrate volume data behind opaque or semi-transparent surfaces. It differs from pure volume rendering by prioritizing surface intersection tests to clip the integration range, and from pure mesh rendering by retaining the ability to render internal volume structures up to the surface boundary [2165-multi-mode-rendering-with-iso-surfaces-and-meshes].

### [Hybrid Isosurface/DVR Rendering](8367-hybrid-isosurface-dvr-rendering.md)
Hybrid Isosurface/DVR Rendering is a technique for viewpoints inside a volume with a wide field of view, such as virtual colonoscopy, that combines surface rendering for the near wall with Direct Volume Rendering (DVR) for structures behind it. It sets itself apart from pure DVR by providing sharp surface detail for the immediate foreground (the colon wall) while preserving the internal texture and transparency of distant structures, which pure isosurface rendering would occlude.

### [Hybrid Implicit Isosurface Ray Tracing](11176-hybrid-implicit-isosurface-ray-tracing.md)
A rendering technique that extracts a list of active vertex-centered cubic primitives (octants or dual cells) from cell-centered TB-AMR data, builds a BVH over them, and performs implicit ray-isosurface intersection within each primitive, avoiding the memory cost of explicit triangle extraction [11176-hybrid-implicit-isosurface-ray-tracing].

### [Mesh-based Neural Rendering](11555-mesh-based-neural-rendering.md)
A rendering approach where the final color of a pixel is determined by the attributes of the intersection point between a ray and the surface of the object, as well as the viewing direction, reformulating the volume rendering integral [11555-mesh-based-neural-rendering].

### Hybrid Volume-Surface Rendering
#### definition
- A rendering strategy combining volume and surface rendering techniques to enable real-time performance on consumer hardware [S1].
- A rendering strategy combining volume rendering with rasterized proxy mesh to boost speed by determining a rough distance to the object [S1].

#### step
- Generate a proxy mesh by applying marching cubes on the reconstructed density field followed by mesh decimation [S1].
- Rasterize the proxy mesh to obtain the hit point on the surface for the central axis of the cone [S1].
- Uniformly sample spheres within distance $\Delta t$ from the hit point in the central axis of the cone, yielding a $2\Delta t$ sampling interval [S1].
- Rasterize the proxy mesh to obtain the hit point on the surface for the central axis of the cone [S1].
- Uniformly sample spheres within a distance of ∆t from the hit point in the central axis of the cone, yielding a 2∆t sampling interval [S1].

#### number
- Achieves 60 FPS on an Nvidia RTX 3060 graphics card [S1].
- Enables real-time rendering at >60 FPS on consumer-level GPUs [S1].

## Background
Hybrid and multi-mode rendering techniques have evolved to address the computational complexity of rendering high-fidelity volumetric data alongside explicit geometry. The integration of surface clipping for volume rendering allows for efficient handling of occlusion, while the combination of isosurface detection with DVR enables realistic visualization of internal structures in medical and scientific imaging. The recent incorporation of neural radiance fields (NeRF) and proxy meshes further extends these methods to real-time applications on consumer hardware, leveraging rasterization for speed and volume rendering for detail [S1, 2165-multi-mode-rendering-with-iso-surfaces-and-meshes, 8367-hybrid-isosurface-dvr-rendering, 11176-hybrid-implicit-isosurface-ray-tracing, 11555-mesh-based-neural-rendering].

## Sources
[S1] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
