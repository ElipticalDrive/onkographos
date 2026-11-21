# Shading and Material Representation

Also called: Isosurface Shading, Surface Property Rendering

## Overview
This family defines how to compute the color and material appearance of an isosurface detected during first-hit ray casting. It solves the problem of visualizing the geometric and material properties of the extracted surface, distinguishing itself from sibling methods that focus on traversal speed or intersection refinement by concentrating on lighting models, differential geometry, and texture mapping. It sets itself apart from volumetric rendering by ignoring contributions from voxels behind the surface, relying instead on local surface properties [S3, unofficial][S5, unofficial].

## Shared principles
The family operates on the intersection point $\mathbf{p}$ found by the parent method. The core mechanism involves:
1.  **Normal Estimation**: Computing the surface normal $\mathbf{n}$ from the gradient of the scalar field $f$ at $\mathbf{p}$, typically $\mathbf{n} = \nabla f(\mathbf{p}) / \|\nabla f(\mathbf{p})\|$ [S3, unofficial][S1, unofficial].
2.  **Lighting Model Application**: Applying a shading model (e.g., Phong, Blinn-Phong) using $\mathbf{n}$, the view direction, and light directions to compute diffuse and specular components [S3, unofficial].
3.  **Property-Based Coloring**: Optionally mapping scalar field derivatives or curvature to color to reveal internal structure or geometric features, rather than using uniform material colors [S1, unofficial].

## Subtopics

### [Deferred Shading for Isosurfaces](867-deferred-shading-for-isosurfaces.md)
[Deferred Shading for Isosurfaces](867-deferred-shading-for-isosurfaces.md) is a multi-pass technique that decouples shading from volume sampling. It computes high-order differential properties (Hessian, principal curvatures) in separate rendering passes using tri-cubic B-spline convolution, allowing for advanced effects like curvature-aligned flow advection and non-photorealistic ridge/valley depiction. Choose this when geometric accuracy and advanced visualization of surface curvature are required, accepting the cost of multiple passes and floating-point texture storage.

### [Isosurface Color Enhancement](7670-isosurface-color-enhancement.md)
[Isosurface Color Enhancement](7670-isosurface-color-enhancement.md) assigns material color based on the accumulated alpha in a semi-transparent neighborhood behind the surface, using the scalar changing rate (directional derivative) as a proxy for neighborhood thickness. It reveals internal structures without changing the ray-traversal scheme. Choose this when you need to visualize density variations or internal features behind the isosurface with minimal computational overhead compared to full volume rendering.

### Matcap Texturing
Texturing technique using a material capture image (matcap) based on the view direction and surface normal, referencing Takikawa et al. 2021 [S1, unofficial].
1.  If the pixel is not collided, set color to white (255, 255, 255) [S1, unofficial].
2.  For collided pixels, create `ray_d_sc` by negating the Z component of the ray direction: `ray_d_sc = (ray_d.x, ray_d.y, -ray_d.z)` [S1, unofficial].
3.  Compute dot product `ray_d_n_dot = nrms.dot(ray_d_sc)` [S1, unofficial].
4.  Compute reflection vector `r = ray_d_sc - nrms * ray_d_n_dot * 2.0` [S1, unofficial].
5.  Compute magnitude `m = 2 * r.norm()` [S1, unofficial].
6.  Compute UV coordinates: `x = r.x / m + 0.5`, `y = r.y / m + 0.5` [S1, unofficial].
7.  Invert UVs: `x = 1 - x`, `y = 1 - y` [S1, unofficial].
8.  Clamp X and Y to [0, 1] [S1, unofficial].
9.  Map to texture coordinates: `x_ = round(x * matcap_img.cols)`, `y_ = round(y * matcap_img.rows)` [S1, unofficial].
10. Sample the matcap image at `(y_, x_)` and assign RGB values to the output image [S1, unofficial].

## Background
The concept of shading isosurfaces derived from scalar fields originated with the development of ray casting techniques for medical and scientific visualization in the late 1980s and early 1990s. Early methods relied on simple Phong shading using finite-difference normals [S3, unofficial]. The use of higher-order derivatives (Hessian, curvature) for shading was introduced to enhance the perception of surface shape and internal structures, with deferred shading approaches becoming prominent in the 2000s as GPU capabilities allowed for multi-pass rendering [S1, unofficial]. Matcap texturing for implicit surfaces has been adapted from polygonal rendering techniques more recently [S1, unofficial].

## Sources
[S1] Leesan-haruki/FunctionRenderer, https://github.com/Leesan-haruki/FunctionRenderer, fetched 2026-10-26 (unofficial)
