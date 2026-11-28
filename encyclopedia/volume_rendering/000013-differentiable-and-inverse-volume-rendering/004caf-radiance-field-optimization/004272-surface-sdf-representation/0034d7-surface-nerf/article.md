# Surface NeRF

Also called: NeRF surf, Surface rendering via NeRF

## Overview
Surface NeRF (SNeRF) converts a pre-trained Neural Radiance Field (NeRF) into a 3D mesh and a surface light field, reducing per-pixel rendering cost by evaluating the NeRF only at the surface isosurface rather than via volumetric raycasting [S1]. It differs from its siblings [NeuS](6060-neus.md), [VolSDF](10715-volsdf.md), and [Instant-NSR](12440-instant-nsr.md) by starting from an existing volume-rendered NeRF and extracting a mesh via isosurface extraction, whereas the siblings learn the SDF and radiance jointly from scratch using differentiable volume rendering [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A pre-trained NeRF network $\mathcal{N}$ mapping 3D position $\mathbf{x}$ and view direction $\mathbf{v}$ to color $\mathbf{c}$ and density $\sigma$, and a set of posed input images [S1].
- **Output:** A refined 3D mesh (isosurface) and a surface light field representation enabling fast rendering [S1].

**Data Structures:**
- **Density Grid:** A regular 3D grid storing density values $\sigma$, initialized by evaluating the pre-trained NeRF at grid vertices [S1].
- **NeRF Network:** The original MLP parameters, finetuned during the optimization process [S1].

**Parameters:**
- **Isosurface Threshold:** 50, as suggested by the original NeRF paper [S1].
- **Optimization Iterations:** 5000 [S1].
- **Samples per Pixel:** 1, compared to 128 in baseline volume rendering [S1].

**Steps:**
1. **Initialization:** Evaluate the pre-trained NeRF on a regular 3D grid to construct the input density field [S1].
2. **Mesh Extraction:** Extract an isosurface from the density field at the fixed threshold (50) to serve as the initial geometry [S1]. This surface is an accurate but noisy model of the subject shape [S1].
3. **Finetuning Setup:** Since the NeRF was trained under the assumption of volume rendering, it must be finetuned to produce good results when evaluated only at the surface [S1].
4. **Optimization Loop:** Perform 5000 iterations of alternating optimization [S1]:
    a. **NeRF Step:** Take one gradient step on the NeRF network while holding the density grid fixed. The optimizer is Adam [S1]. The loss is the original NeRF L2 loss between rendered colors and input images [S1].
    b. **Grid Step:** Take one gradient step on the density grid while holding the NeRF network fixed. The optimizer is standard gradient descent [S1].
5. **Rendering:** Render the isosurface using Rasterize-then-Splat (RtS) [S1]. The NeRF network produces shaded colors $\mathbf{C}_k$ for the surface points, effectively acting as a surface light field [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_nerf_optimization(pretrained_nerf, input_images, grid_resolution, threshold=50, iterations=5000, rays=None):
    # Step 1: Initialize density grid by evaluating NeRF at grid vertices
    density_grid = evaluate_nerf_on_grid(pretrained_nerf, grid_resolution)
    
    # Step 2: Extract initial isosurface at threshold
    mesh = extract_isosurface(density_grid, threshold)
    
    # Initialize optimizers
    nerf_optimizer = Adam(pretrained_nerf.parameters())
    grid_optimizer = SGD(density_grid.parameters())
    
    for i in range(iterations):
        # Step 4a: Optimize NeRF (hold grid fixed)
        nerf_optimizer.zero_grad()
        rendered_colors = render_surface_light_field(pretrained_nerf, mesh, rays)
        loss = compute_l2_loss(rendered_colors, input_images)
        loss.backward()
        nerf_optimizer.step()
        
        # Step 4b: Optimize Density Grid (hold NeRF fixed)
        grid_optimizer.zero_grad()
        # [S1] The source does not specify the exact gradient computation for the grid;
        # it states to optimize the density grid directly via gradients from rendering.
        loss.backward()
        grid_optimizer.step()
        
    return mesh, pretrained_nerf

def evaluate_nerf_on_grid(nerf, resolution):
    grid = np.zeros((resolution, resolution, resolution))
    for x in range(resolution):
        for y in range(resolution):
            for z in range(resolution):
                pos = grid_to_world(x, y, z, resolution)
                grid[x, y, z] = nerf(pos, None).density
    return grid

def extract_isosurface(grid, threshold):
    # [G] Standard Marching Cubes algorithm extracts vertices and faces from a scalar field at the threshold
    return marching_cubes(grid, threshold)

def render_surface_light_field(nerf, mesh, rays):
    colors = []
    for ray in rays:
        point, normal = ray_mesh_intersect(ray, mesh)
        if point is not None:
            view_dir = ray.direction
            color = nerf(point, view_dir).color
            colors.append(color)
        else:
            colors.append(np.zeros(3))
    return np.array(colors)

def ray_mesh_intersect(ray, mesh):
    # [G] Standard ray-triangle intersection returns closest hit or (None, None)
    closest_t = np.inf
    closest_point = None
    closest_normal = None
    for triangle in mesh.faces:
        t, hit_point = intersect_ray_triangle(ray.origin, ray.direction, triangle)
        if 0 < t < closest_t:
            closest_t = t
            closest_point = hit_point
            closest_normal = triangle.normal
    return closest_point, closest_normal

def compute_l2_loss(rendered, target):
    return np.mean((rendered - target) ** 2)

def grid_to_world(x, y, z, resolution):
    # [G] Maps integer grid coordinates to normalized 3D space [-1, 1]^3
    return np.array([x / (resolution - 1) * 2 - 1, y / (resolution - 1) * 2 - 1, z / (resolution - 1) * 2 - 1])

def intersect_ray_triangle(origin, direction, triangle):
    # [G] Möller–Trumbore algorithm
    v1, v2, v3 = triangle.vertices
    edge1 = v2 - v1
    edge2 = v3 - v1
    h = np.cross(direction, edge2)
    a = np.dot(edge1, h)
    if -0.00001 < a < 0.00001:
        return np.inf, None
    f = 1.0 / a
    s = origin - v1
    u = f * np.dot(s, h)
    if u < 0.0 or u > 1.0:
        return np.inf, None
    q = np.cross(s, edge1)
    v = f * np.dot(direction, q)
    if v < 0.0 or u + v > 1.0:
        return np.inf, None
    t = f * np.dot(edge2, q)
    if t > 0.00001:
        return t, origin + t * direction
    return np.inf, None
```

**Edge Cases and Pitfalls:**
- **Optimizer Instability:** Adam is unstable for optimizing the density grid because the isosurface does not constrain all grid values at each iteration [S1]. Standard gradient descent is used for the grid instead [S1].
- **Complex Geometry:** On complex scenes like Fern, the extremely complex isosurface limits quality improvement over the fixed surface baseline [S1]. The method loses 0.9 PSNR compared to full volume rendering on such scenes [S1].
- **Closed-Form Requirement:** The method requires a closed-form evaluation function for the surface, which may not be available for all surface representations (e.g., some subdivision schemes) [S1].
- **Occlusion Handling:** The splatting step currently assumes a single surface at each pixel, which may cause artifacts in highly occluded regions [S1].

## Operating characteristics
- **Computational Cost:** Reduces per-pixel computational cost from 226 MFLOPS (volume rendering) to 1.7 MFLOPS (SNeRF) [S1]. This achieves a runtime cost reduction of more than 100× compared to baseline NeRF rendering [S1].
- **Quality:** On the Lego dataset, SNeRF achieves a PSNR of 29.61, which is within 0.3 of the baseline NeRF PSNR of 29.91 [S1]. It improves 2.2 PSNR over the baseline without surface optimization (SNeRF fixed, PSNR 27.44) [S1]. On the Fern dataset, SNeRF achieves a PSNR of 23.52, losing 0.9 PSNR to full volume rendering (24.38) [S1]. It improves only 0.1 PSNR over the fixed surface baseline (23.41) [S1].
- **Strengths:** Significant reduction in rendering time; works with any surface that can be expressed as a non-differentiable sampling and a differentiable evaluation function [S1].
- **Limitations:** Quality degradation on scenes with extremely complex isosurfaces; requires a closed-form evaluation function; assumes a single surface at each pixel during splatting [S1].

## Background
Surface NeRF was introduced in the paper "Differentiable Surface Rendering via Non-Differentiable Sampling" [S1]. It builds on the original NeRF work [S1] and aims to address the long rendering times associated with volumetric raycasting by converting the volume representation into a surface representation [S1]. The method leverages the Rasterize-then-Splat (RtS) technique for differentiable surface rendering [S1].

## Sources
[S1] Differentiable Surface Rendering via Non-Differentiable Sampling, https://doi.org/10.1109/iccv48922.2021.00603, fetched 2026-10-26
