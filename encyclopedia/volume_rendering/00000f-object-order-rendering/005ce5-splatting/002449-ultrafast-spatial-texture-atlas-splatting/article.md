# Ultrafast Spatial Texture-Atlas Splatting

Also called: Texture-Atlas Splatting

## Overview
Ultrafast Spatial Texture-Atlas Splatting is a learned sparse scene representation that eliminates runtime neural network inference by baking view-independent high-frequency texture into a hardware-accelerated 2D texture atlas [S1, unofficial]. It represents the scene as a sparse set of oriented 2D Beta kernels, separating low-frequency view-dependent appearance (stored per-surfel) from high-frequency spatial texture (stored in the atlas) [S1, unofficial]. This approach achieves 7–9× faster rendering than 3D Gaussian Splatting (3DGS) on common benchmarks by reducing overdraw via flat Beta kernels and replacing expensive ray-surfel intersection shaders with an exact rational reformulation [S1, unofficial].

## How it works
**Inputs:** A set of optimized 2D surfels (position $\boldsymbol{\mu}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $o_i$, and view-dependent color parameters) and a trained multi-resolution hash grid $E_\theta$ with MLP decoder $f_\theta$ [S1, unofficial].
**Outputs:** A compressed RGB texture atlas (55–320 MB for typical scenes) and the surfel attributes [S1, unofficial]. The final rendered image.

**Data Structures:**
1.  **Surfel Attributes:** For each surfel $i$:
    *   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Position.
    *   $\mathbf{q}_i \in \mathbb{R}^4$: Rotation (quaternion).
    *   $\mathbf{s}_i \in \mathbb{R}^2$: 2D scale.
    *   $o_i \in \mathbb{R}$: Opacity.
    *   $f_{SV}(\mathbf{d})$: View-dependent color function parameters (stored per-surfel) [S1, unofficial].
2.  **Texture Atlas:** A 2D RGB texture where each surfel $i$ maps to a rectangular region [S1, unofficial]. The texels store the signed residual color $\rho(\mathbf{x}) = f_\theta(E_\theta(\mathbf{x}))$ [S1, unofficial]. The atlas is allocated once after sparsification [S1, unofficial].

**Numbered Steps:**

1.  **Primitive Representation:** Represent the scene as a sparse set of oriented 2D surfels [S1, unofficial]. Each surfel $i$ is defined by its position, rotation, scale, and opacity [S1, unofficial].
2.  **Kernel Selection:** Use the Deformable Beta kernel $\mathcal{B}(x;b)$ instead of a Gaussian [S1, unofficial]. The kernel shape is controlled by a parameter $b \in \mathbb{R}$:
    $$ \mathcal{B}(x;b) = (1-x)^{\beta(b)} $$
    where $x \in [0,1]$ is the squared radial distance from the kernel center normalized to 1 at the edge of the support, and $\beta(b) = 5\sigma(b)$ with $\sigma$ being the sigmoid function [S1, unofficial]. Small $b$ values produce flat, hard-edged disks to reduce overdraw; large $b$ values produce soft, Gaussian-like profiles [S1, unofficial]. The parameter $b$ is restricted to lie between a flat disk and a soft Gaussian [S1, unofficial].
3.  **Appearance Decomposition:** Decompose the fragment color $\mathbf{c}(\mathbf{x}, \mathbf{d})$ into a per-surfel view-dependent term and a view-independent residual:
    $$ \mathbf{c}(\mathbf{x}, \mathbf{d}) = \text{ReLU}\left(\text{ReLU}(f_{SV}(\mathbf{d}) + b_{SV}) + f_\theta(\mathbf{E}_\theta(\mathbf{x}))\right) $$
    where $b_{SV} = 0.5$ is a fixed offset providing a neutral gray baseline, $f_{SV}$ is the per-surfel view-dependent color, and $f_\theta(\mathbf{E}_\theta(\mathbf{x}))$ is the residual from the hash grid $E_\theta$ and MLP decoder $f_\theta$ [S1, unofficial].
4.  **Training:**
    *   Query the hash grid at every fragment along the surfel's surface during training to absorb fine spatial texture [S1, unofficial].
    *   Per-surfel attributes remain constant across the surfel to absorb low-frequency view-dependent appearance [S1, unofficial].
    *   Use signed texture residuals (positive and negative color offsets) to provide representational power, allowing a single surfel to reproduce color variation that positive-only textures cannot [S1, unofficial].
    *   Train jointly with the multi-view error criterion of FastGS to limit densification [S1, unofficial].
    *   Introduce a falloff prior using Beta kernels to regularize kernels toward flat, hard-edged profiles, dampening positional gradients and reducing primitive count [S1, unofficial].
5.  **Baking:**
    *   Sample the hash-grid across the surface of each primitive during training [S1, unofficial].
    *   Bake the output of the hash-grid sampling into a compressed RGB texture atlas [S1, unofficial].
    *   Finetune the atlas texels directly against the training images to overcome hash collisions from training [S1, unofficial].
6.  **Rendering:**
    *   Perform fast, per-fragment texture fetches from the atlas [S1, unofficial].
    *   Evaluate the ray-surfel intersection using an exact rational reformulation of the intersection test [S1, unofficial].
    *   Composite the surfels in front-to-back order using alpha blending [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

def beta_kernel(x, b):
    """Compute the Deformable Beta kernel value."""
    beta = 5.0 * sigmoid(b)
    x = max(0.0, min(1.0, x))
    return (1.0 - x) ** beta

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def normalize(v):
    length = math.sqrt(dot(v, v))
    return [v[0]/length, v[1]/length, v[2]/length]

def project_screen_bounds(surfel, camera):
    # Standard projection of surfel bounding box to screen coordinates
    # Returns object with x_min, x_max, y_min, y_max
    pass

def intersect_ray_plane(ray_origin, ray_dir, surfel):
    # Exact rational reformulation of ray-surfel intersection
    # Returns t, u, v parameters
    pass

def transform_to_local(surfel, world_pos):
    # Transforms world position to surfel's local 2D coordinate system
    pass

def render_scene(surfels, camera, texture_atlas):
    # Initialize output image with zeros
    image = [[0.0, 0.0, 0.0] for _ in range(camera.height * camera.width)]
    
    # Sort surfels front-to-back by depth along view direction
    sorted_surfels = sorted(surfels, key=lambda s: dot(s.position, camera.view_direction))
    
    for surfel in sorted_surfels:
        screen_bounds = project_screen_bounds(surfel, camera)
        
        for y in range(screen_bounds.y_min, screen_bounds.y_max):
            for x in range(screen_bounds.x_min, screen_bounds.x_max):
                # Exact rational ray-surfel intersection test
                ray_origin = camera.center
                ray_dir = normalize(camera.screen_to_ray(x, y) - ray_origin)
                t, u, v = intersect_ray_plane(ray_origin, ray_dir, surfel)
                
                if t > 0 and 0.0 <= u <= 1.0 and 0.0 <= v <= 1.0:
                    # Compute normalized squared distance from surfel center
                    local_pos = transform_to_local(surfel, ray_origin + t * ray_dir)
                    dist_sq = dot(local_pos, local_pos)
                    
                    kernel_val = beta_kernel(dist_sq, surfel.b_param)
                    alpha = surfel.opacity * kernel_val
                    
                    if alpha > 0.0:
                        # Sample baked texture atlas at local UV coordinates
                        residual = texture_atlas.sample(u, v)
                        
                        # Compute view-dependent color
                        f_sv = surfel.compute_view_dependent_color(ray_dir)
                        b_sv = 0.5
                        base_color = max(0.0, f_sv + b_sv)
                        final_color = max(0.0, base_color + residual)
                        
                        # Front-to-back alpha compositing
                        idx = y * camera.width + x
                        c = image[idx]
                        image[idx] = [
                            c[0] + (1.0 - c[0]) * final_color[0] * alpha,
                            c[1] + (1.0 - c[1]) * final_color[1] * alpha,
                            c[2] + (1.0 - c[2]) * final_color[2] * alpha
                        ]
    return image
```

**Parameters:**
*   $b_{SV} = 0.5$: Fixed offset inside the inner ReLU activation [S1, unofficial].
*   Texture atlas size: 55–320 MB for typical scenes [S1, unofficial].
*   $\beta(b) = 5\sigma(b)$: Shape parameter scaling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** Hash collisions from training can degrade quality. Finetuning the atlas texels directly against training images mitigates this [S1, unofficial].
*   **Overdraw:** Soft Gaussian tails cause redundant overlaps. Using flat Beta kernels (small $b$) reduces overdraw and primitive count [S1, unofficial].
*   **Sparsity:** Pipelines using per-primitive textures start from a converged dense model with densification frozen, capping sparsity. This method allocates the atlas once after sparsification, allowing higher sparsity [S1, unofficial].

## Subtopics

### Deformable Beta Kernels
A kernel family defined by $\mathcal{B}(x;b) = (1-x)^{\beta(b)}$ that interpolates between flat disks and soft Gaussians via the parameter $b$ [S1, unofficial]. Chosen to reduce overdraw compared to standard Gaussians [S1, unofficial].

### Texture Atlas Baking
The process of converting view-independent neural queries (hash grid + MLP) into a static 2D texture map [S1, unofficial]. Enables hardware-accelerated sampling without runtime neural inference [S1, unofficial].

### Dynamic Inference for Texture Maps
When a block requiring rendering is encountered, the UE4 submodule supplies the block and level index information, while the inference submodule provides corresponding regional meshes and feature maps [S2]. If feature maps are not ready, inference is carried out to generate them [S2]. Uses dynamic inference instead of storing all texture maps to reduce storage overhead for large-scale scenes [S2].

## Operating characteristics
*   **Time Complexity:** Rendering is dominated by texture fetches and simple arithmetic, avoiding neural network evaluations [S1, unofficial]. The rational intersection test is computationally cheaper than standard ray-surfel shaders [S1, unofficial].
*   **Memory Complexity:** Requires a texture atlas of 55–320 MB for typical scenes, which is larger than memory-optimized 3DGS representations but fits on low-to-mid-range GPUs [S1, unofficial].
*   **Performance:** Renders 7.1–9.1× faster than 3DGS and 1.2–1.8× faster than FastGS and Speedy-Splat on common benchmarks [S1, unofficial].
*   **Strengths:** High rendering speed due to elimination of runtime neural inference; reduced overdraw via flat kernels; high sparsity [S1, unofficial].
*   **Limitations:** Larger memory footprint than some compressed 3DGS variants; requires training to bake the atlas; signed residuals require careful handling of negative values in texture formats [S1, unofficial].
*   **When to Use:** Real-time rendering of high-fidelity scenes where GPU compute is limited by neural network inference overhead [S1, unofficial].
*   **When Not to Use:** When memory is extremely constrained and smaller scene representations are prioritized over rendering speed [S1, unofficial].

## Background
Ultrafast Spatial Texture-Atlas Splatting was introduced by Petkov and Klaus from Siemens Healthineers in 2026 [S1, unofficial]. It builds on 2D Gaussian Splatting (2DGS) and FastGS, aiming to solve the rendering overhead caused by runtime neural network evaluations in neural radiance representations [S1, unofficial]. The method leverages the observation that view-independent high-frequency texture can be baked into a texture atlas, while view-dependent low-frequency appearance remains per-surfel [S1, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
[S2] UE4-NeRF: Neural Radiance Field for Real-Time Rendering of Large-Scale Scene, https://doi.org/10.48550/arXiv.2310.13263, fetched 2026-10-29
