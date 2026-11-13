# Surface NeRF

Also called: NeRF surf, Surface rendering via NeRF

## Overview
Surface NeRF (SNeRF) is a method that converts a pre-trained Neural Radiance Field (NeRF) into a 3D mesh and a surface light field, reducing per-pixel rendering cost by evaluating the NeRF only at the surface isosurface rather than via volumetric raycasting [S1]. It differs from its siblings [NeuS](6060-neus.md), [VolSDF](10715-volsdf.md), and [Instant-NSR](12440-instant-nsr.md) by starting from an existing volume-rendered NeRF and extracting a mesh via isosurface extraction, whereas the siblings learn the SDF and radiance jointly from scratch using differentiable volume rendering [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A pre-trained NeRF network $\mathcal{N}$ (mapping 3D position $\mathbf{x}$ and view direction $\mathbf{v}$ to color $\mathbf{c}$ and density $\sigma$) and a set of posed input images [S1].
- **Output:** A refined 3D mesh (isosurface) and a surface light field representation that allows for fast rendering [S1].

**Data Structures:**
- **Density Grid:** A regular 3D grid storing density values $\sigma$. The grid is initialized by evaluating the pre-trained NeRF at grid vertices [S1].
- **NeRF Network:** The original MLP parameters, finetuned during the process [S1].

**Parameters:**
- **Isosurface Threshold:** 50, as suggested by the original NeRF paper [S1].
- **Optimization Iterations:** 5000 [S1].
- **Samples per Pixel:** 1 (compared to 128 in baseline volume rendering) [S1].

**Steps:**
1. **Initialization:** Evaluate the pre-trained NeRF on a regular 3D grid to construct the input density field [S1].
2. **Mesh Extraction:** Extract an isosurface from the density field at the fixed threshold (50) to serve as the initial geometry [S1]. This surface is an accurate but noisy model of the subject shape [S1].
3. **Finetuning Setup:** Since the NeRF was trained under the assumption of volume rendering, it must be finetuned to produce good results when evaluated only at the surface [S1].
4. **Optimization Loop:** Perform 5000 iterations of alternating optimization:
    a. **NeRF Step:** Take one gradient step on the NeRF network while holding the density grid fixed. The optimizer is Adam [S1]. The loss is the original NeRF L2 loss between rendered colors and input images [S1].
    b. **Grid Step:** Take one gradient step on the density grid while holding the NeRF network fixed. The optimizer is standard gradient descent [S1].
5. **Rendering:** Render the isosurface using Rasterize-then-Splat (RtS) [S1]. The NeRF network produces shaded colors $\mathbf{C}_k$ for the surface points, effectively acting as a surface light field [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_nerf_optimization(pretrained_nerf, input_images, grid_resolution, threshold=50, iterations=5000):
    # Step 1: Initialize density grid by evaluating NeRF
    density_grid = evaluate_nerf_on_grid(pretrained_nerf, grid_resolution)
    
    # Step 2: Extract initial isosurface
    mesh = extract_isosurface(density_grid, threshold)
    
    # Initialize optimizers
    nerf_optimizer = Adam(pretrained_nerf.parameters())
    grid_optimizer = SGD(density_grid.parameters()) # Standard gradient descent
    
    for i in range(iterations):
        # Step 4a: Optimize NeRF (hold grid fixed)
        nerf_optimizer.zero_grad()
        # Render using the current mesh and NeRF
        rendered_colors = render_surface_light_field(pretrained_nerf, mesh)
        # Compute L2 loss against input images
        loss = compute_l2_loss(rendered_colors, input_images)
        loss.backward()
        nerf_optimizer.step()
        
        # Step 4b: Optimize Density Grid (hold NeRF fixed)
        grid_optimizer.zero_grad()
        # Re-evaluate NeRF at grid points to get density gradients
        # Note: The source implies optimizing the grid values directly
        # based on the rendering loss, likely via differentiable rasterization
        # or a surrogate loss related to the surface position.
        # The source states: "optimize the density grid while finetuning the NeRF"
        # and "Adam is unstable for grid optimization".
        # We assume a differentiable rendering pass or a specific grid loss.
        # Given the source text, we perform a gradient step on the grid.
        # The exact gradient source for the grid is not detailed beyond "optimize the density grid".
        # [G] In practice, this often involves backpropagating through the mesh extraction 
        # or using a signed distance field loss, but the source specifies optimizing the 
        # density grid values directly. We assume a differentiable renderer or 
        # a specific grid-update rule is applied here.
        # To remain faithful to the source "perform one gradient step on the density grid",
        # we assume the rendering pipeline provides gradients w.r.t the grid.
        loss.backward() # Assuming loss is recomputed or gradients are passed
        grid_optimizer.step()
        
    return mesh, pretrained_nerf

def render_surface_light_field(nerf, mesh):
    # Step 5: Render using RtS
    # Splat the mesh vertices/faces
    # For each pixel, evaluate NeRF at the surface intersection
    colors = []
    for pixel in pixels:
        # Find intersection with mesh
        point, normal = ray_mesh_intersect(pixel.ray, mesh)
        if point is not None:
            # Evaluate NeRF at surface point
            color = nerf(point, normal.view_direction)
            colors.append(color)
    return colors
```

**Edge Cases and Pitfalls:**
- **Optimizer Instability:** Adam is unstable for optimizing the density grid because the isosurface does not constrain all grid values at each iteration [S1]. Therefore, standard gradient descent is used for the grid [S1].
- **Complex Geometry:** On complex scenes like Fern, the extremely complex isosurface limits quality improvement over the fixed surface baseline [S1]. The method loses 0.9 PSNR compared to full volume rendering on such scenes [S1].
- **Closed-Form Requirement:** The method requires a closed-form evaluation function for the surface, which may not be available for all surface representations (e.g., some subdivision schemes) [S1].

## Operating characteristics
- **Computational Cost:** Reduces per-pixel computational cost from 226 MFLOPS (volume rendering) to 1.7 MFLOPS (SNeRF) [S1]. This is a reduction of more than 100x in runtime cost compared to baseline NeRF rendering [S1].
- **Quality:** On the Lego dataset, SNeRF achieves a PSNR of 29.61, which is within 0.3 of the baseline NeRF PSNR of 29.91 [S1]. It improves 2.2 PSNR over the baseline without surface optimization (SNeRF fixed, PSNR 27.44) [S1]. On the Fern dataset, SNeRF achieves a PSNR of 23.52, losing 0.9 PSNR to full volume rendering (24.38) [S1]. It improves only 0.1 PSNR over the fixed surface baseline (23.41) [S1].
- **Strengths:** Significant reduction in rendering time; works with any surface that can be expressed as a non-differentiable sampling and a differentiable evaluation function [S1].
- **Limitations:** Quality degradation on scenes with extremely complex isosurfaces; requires a closed-form evaluation function; assumes a single surface at each pixel during splatting [S1].

## Background
Surface NeRF was introduced in the paper "Differentiable Surface Rendering via Non-Differentiable Sampling" [S1]. It builds on the original NeRF work [S1] and aims to address the long rendering times associated with volumetric raycasting by converting the volume representation into a surface representation [S1]. The method leverages the Rasterize-then-Splat (RtS) technique for differentiable surface rendering [S1].

## Sources
[S1] Differentiable Surface Rendering via Non-Differentiable Sampling, https://doi.org/10.1109/iccv48922.2021.00603, fetched 2026-10-26
