# SplatFace


## Overview
SplatFace is a Gaussian model initialization and optimization strategy designed for high-fidelity head avatar reconstruction from few input views [S1, unofficial]. It addresses the challenge of initializing 3D Gaussians on complex surfaces by placing them on a template mesh and jointly optimizing the Gaussian parameters and the mesh geometry using a splat-to-mesh distance loss [S1, unofficial].

## How it works
**Inputs:** A template mesh, a set of input images (few-view), and camera poses [G].
**Outputs:** A set of 3D Gaussians and an optimized mesh geometry [G].

**Data Structures:**
1. **Template Mesh:** Defined by vertices $V$ and faces $F$ [G].
2. **Gaussian Set:** A list of $N$ Gaussians, each with position $\mathbf{p}$, covariance matrix $\mathbf{\Sigma}$, opacity $\alpha$, and spherical harmonic coefficients for color [G].

**Steps:**
1. **Initialization:** Initialize 3D Gaussians on the surface of the template mesh [S1, unofficial].
2. **Rendering:** Render the scene using standard 3D Gaussian Splatting rasterization to produce an image [G].
3. **Loss Calculation:** Compute a loss function comprising an image loss (difference between rendered and input images, e.g., L1 or SSIM) [G] and a splat-to-mesh distance loss that penalizes the distance between the splatted Gaussians and the current mesh surface [S1, unofficial].
4. **Joint Optimization:** Update both the Gaussian parameters and the mesh vertices using gradient descent [S1, unofficial].

**Formulas:**
The total loss $L$ is:
$$ L = L_{\text{image}} + \lambda L_{\text{splat-mesh}} $$
where $L_{\text{splat-mesh}}$ measures the distance between the Gaussian splats and the mesh surface [S1, unofficial].

**Parameters:**
* $\lambda$: Weight for the splat-to-mesh distance loss. Value not given in the sources.

**Edge Cases:**
* **Degenerate Mesh:** If the template mesh has poor topology, initialization may be inaccurate [G].
* **Precision:** Joint optimization requires careful balancing of learning rates for Gaussians and mesh vertices [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard helper functions
def alpha_blend(gaussians, camera_poses):
    # Sort Gaussians by depth, project to 2D, and alpha-composite [G]
    return composite_projected_gaussians(gaussians, camera_poses)

def distance_point_to_triangle(p, V, F):
    # Compute Euclidean distance from point p to each triangle in F [G]
    return min(euclidean_dist(p, tri) for tri in triangles(V, F))

# Inputs: mesh_V, mesh_F, images, camera_poses, num_steps, lambda_weight, lr
# 1. Initialize Gaussians on template mesh [S1, unofficial]
gaussians = []
for face in mesh_F:
    u, v = random(), random()
    if u + v > 1: u, v = 1 - u, 1 - v
    pos = (1 - u - v) * mesh_V[face[0]] + u * mesh_V[face[1]] + v * mesh_V[face[2]]
    gaussians.append({'pos': pos, 'covar': identity(3), 'alpha': 1.0, 'color': sh_coeffs()})

# 2-4. Joint optimization [S1, unofficial]
for step in range(num_steps):
    rendered_image = alpha_blend(gaussians, camera_poses)
    loss_image = L1(rendered_image, input_image)
    
    loss_splat_mesh = 0.0
    for g in gaussians:
        loss_splat_mesh += distance_point_to_triangle(g['pos'], mesh_V, mesh_F)
        
    total_loss = loss_image + lambda_weight * loss_splat_mesh
    total_loss.backward()
    
    # Explicit gradient descent update [G]
    for param in all_parameters:
        if param.grad is not None:
            param.data -= lr * param.grad
```

## Subtopics

### [MeshInitializer](19073-meshinitializer.md)
MeshInitializer initializes 3D Gaussian splats by placing them on the surface of a triangle mesh using barycentric coordinates to determine position, normal interpolation, and color interpolation. It differs from VoxelInitializer by adhering strictly to the mesh topology rather than a grid, and from generic point-cloud initialization by leveraging explicit geometric surface data. It is a variant of Gaussian Splatting Initialization and builds on [SplatFace](17294-splatface.md).

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.
**Measured Performance:** Not given in the sources.
**Scaling:** Not given in the sources.
**Strengths:** Not given in the sources.
**Limitations:** Not given in the sources.
**When to use:** Not given in the sources.
**When not:** Not given in the sources.

## Background
SplatFace was introduced in reference [232] as part of the recent advances in 3D Gaussian Splatting for avatar reconstruction [S1, unofficial]. It is cited in the context of synthesizing high-quality avatars from few input views, preceding works like GauMesh [233] and others in the field [S1, unofficial].

## Sources
[S1] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
