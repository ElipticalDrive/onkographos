# Mesh-based Initialization

Also called: Mesh-to-Gaussian Initialization, Surface Sampling Initialization

## Overview
Mesh-based initialization converts a polygonal mesh into a set of 3D Gaussian splats to serve as the starting point for optimization or rendering [G]. It establishes a geometric prior for Gaussian Splatting when a high-quality surface representation is already available, leveraging the explicit nature of Gaussians to support user-guided generation [S1] or scene-level inverse rendering [S2]. It differs from sibling methods like [Angle-pose Cuboid Uniform Initialization (ACUI)](4503-angle-pose-cuboid-uniform-initialization-acui.md) by deriving positions from continuous surface geometry rather than volumetric bounding boxes, and from [Learning-based and Generative Initialization](12356-learning-based-and-generative-initialization.md) by relying on deterministic geometric sampling rather than neural priors [G].

## Shared principles
The core mechanism involves uniformly sampling points from the surface of an input mesh and assigning Gaussian parameters to these points [S1, S2]. The position of each Gaussian is determined by the sampled surface coordinates [S1, S2]. The orientation and scale are typically aligned with the local surface geometry (e.g., normal-aligned, thin along the normal) to represent the surface as a collection of flat or slightly thickened primitives [G]. This approach exploits the explicit nature of 3D Gaussian Splatting, allowing for direct integration with existing mesh processing pipelines [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import numpy as np

def initialize_gaussians_from_mesh(mesh, points_per_face):
    """
    Samples points from a mesh surface and initializes 3D Gaussian parameters.
    
    Args:
        mesh: Triangle mesh object with attributes .vertices (N, 3) and .faces (M, 3).
        points_per_face: Number of points to sample per triangle.
        
    Returns:
        means: Array of shape (N_total, 3) for Gaussian centers.
        covariances: Array of shape (N_total, 3, 3) for Gaussian covariances.
        opacities: Array of shape (N_total,) for Gaussian opacities.
        colors: Array of shape (N_total, 3) for Gaussian colors.
    """
    means, covs, opacities, colors = [], [], [], []
    
    for face_idx in range(len(mesh.faces)):
        v0, v1, v2 = mesh.vertices[mesh.faces[face_idx]]
        for _ in range(points_per_face):
            # Uniform sampling on triangle using barycentric coordinates
            r1, r2 = random.random(), random.random()
            if r1 + r2 > 1:
                r1, r2 = 1 - r1, 1 - r2
            u, v = r1, r2
            point = u * v0 + v * v1 + (1 - u - v) * v2
            means.append(point)
            
            # Compute surface normal for orientation
            normal = np.cross(v1 - v0, v2 - v0)
            normal = normal / np.linalg.norm(normal)
            
            # Initialize covariance: isotropic in tangent plane, thin along normal
            scale = 0.01  # [G] Typical initialization scale
            cov = np.eye(3) * scale**2
            cov[2, 2] = scale**2 * 0.1
            # Rotate covariance to align local z-axis with surface normal
            R = _align_z_to_vector(normal)
            cov = R @ cov @ R.T
            covs.append(cov)
            
            opacities.append(1.0)
            colors.append(mesh.face_color(face_idx))
            
    return np.array(means), np.array(covs), np.array(opacities), np.array(colors)

def _align_z_to_vector(v):
    """Constructs a rotation matrix that aligns the z-axis with vector v."""
    z = v / np.linalg.norm(v)
    x = np.array([1.0, 0.0, 0.0])
    if np.abs(np.dot(z, x)) > 0.999:
        x = np.array([0.0, 1.0, 0.0])
    x = np.cross(z, x)
    x = x / np.linalg.norm(x)
    y = np.cross(z, x)
    return np.column_stack((x, y, z))
```

## Subtopics

### [Gaussian-Centric Binding](19104-gaussian-centric-binding.md)
A training-free strategy for associating 3D Gaussians with a proxy mesh to enable deformation transfer, eliminating the need for retraining and reducing sensitivity to mesh topology defects. Unlike mesh-centric strategies that require an "initialization-then-training" workflow with geometric constraints, this method leverages existing training camera rays to bind Gaussians directly to mesh faces via spatial association. It is chosen when unified manipulation of mesh and 3DGS is required without retraining costs or topological sensitivity. [Gaussian-Centric Binding](19104-gaussian-centric-binding.md)

### User-Guided Generation Initialization
- **Definition**: A technique for initializing 3D Gaussian Splatting from user-provided 3D shapes to guide text-to-3D generation, leveraging the explicit nature of Gaussians [S1].
- **Step**: Generate initial points by uniformly sampling points on the surface of the provided user mesh [S1].
- **Parameter**: Set the position learning rate to a relatively small value to better preserve the user's desired geometric shape during optimization [S1].
- **Comparison**: Outperforms Latent-NeRF and Fantasia3D in both geometry and texture quality while mostly maintaining the user-provided geometric prior [S1].

### GeoSplatting with Existing Mesh Initialization
- **Definition**: An extension to GeoSplatting that initializes the optimization process with an existing triangle mesh (e.g., from NeuS or GOF) instead of building geometry from scratch, circumventing isosurface resolution limits and mask dependencies [S2].
- **Step**: Extract a triangle mesh from multi-view images using methods like NeuS or GOF [S2]. Convert the initial mesh into Gaussian points via MGadapter [S2]. Follow the remaining steps of the standard GeoSplatting pipeline [S2]. Refine the initial mesh through gradients derived from 3DGS rendering, leveraging the differentiable nature of MGadapter [S2].
- **Pitfall**: This two-stage extension is not applicable to specular objects because surface reconstruction methods lack effective PBR modeling, leading to noisy meshes that GeoSplatting cannot rectify [S2].
- **Comparison**: Unlike the standard pipeline which uses isosurfaces (limited by resolution and mask dependencies), this method extends applicability to scene-level inverse rendering and avoids those specific limitations [S2].

### Mesh-to-Splat Conversion Pipeline
- **Definition**: A pipeline that loads a polygonal mesh and converts it into a set of 3D Gaussian splats for rendering [S3, unofficial].
- **Step**: Invoke `mesh_to_splat(mesh, params.options, progress_callback)` to generate the Gaussian point cloud [S3, unofficial]. Iterate over output targets and save each generated splat using `saveSplat` with `RadExportMode::Stream` and SSOG options [S3, unofficial].
- **Pitfall**: If the input file is not a mesh, or if `mesh_to_splat` returns an error, log the failure and abort the pipeline [S3, unofficial].

## Background
Mesh-based initialization has emerged as a critical component in advanced Gaussian Splatting applications where geometric priors are valuable [G]. It was notably adopted in works such as G SGEN for user-guided text-to-3D generation [S1] and GeoSplatting for physically-based inverse rendering [S2]. The approach allows for the integration of traditional computer graphics assets (meshes) with differentiable rendering frameworks, enabling hybrid workflows [G].

## Sources
[S1] Text-to-3D using Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.02022, fetched 2026-10-26
[S2] GeoSplatting: Towards Geometry Guided Gaussian Splatting for Physically-Based Inverse Rendering, https://doi.org/10.1109/ICCV51701.2025.02692, fetched 2026-10-30
[S3] MrNeRF/LichtFeld-Studio, https://github.com/MrNeRF/LichtFeld-Studio, fetched 2026-10-27 (unofficial)
