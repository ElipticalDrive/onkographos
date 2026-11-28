# Silhouette-based Mesh Optimization

Also called: Shape Optimization Experiment

## Overview
Silhouette-based Mesh Optimization is a differentiable rendering technique that optimizes a mesh's geometry to match the 2D silhouette of a target object. It solves the problem of shape reconstruction from 2D projections by treating mesh vertices as differentiable parameters. It is distinguished from sibling methods like [Simplified Marching Cubes](3759-simplified-marching-cubes.md) by operating directly on explicit triangle meshes rather than voxel grids, and from [Modified Marching Cubes with Triangle Decimation](3903-modified-marching-cubes-with-triangle-decimation.md) by using gradient-based optimization rather than topological simplification rules.

## How it works
**Inputs:**
- Target silhouette images (e.g., from a hard renderer) of a target object (e.g., airplane) from 24 azimuths.
- Initial mesh geometry (typically a sphere).
- Differentiable renderer configuration (distribution, T-conorm, temperature).

**Outputs:**
- Optimized mesh vertices.

**Data Structures:**
- Mesh: List of vertices $V$ and indices $I$. Vertices are optimized tensors.
- Renderer: Configured with a probability distribution $F$, a T-conorm, and a temperature $\tau$.

**Parameters:**
- Optimizer: Adam with $\beta_1 = 0.5, \beta_2 = 0.95$ [S1].
- Learning rates: $\lambda \in \{10^{-1.25}, 10^{-1.5}, 10^{-1.75}\}$ [S1].
- Temperatures: $\tau \in \{10^{-0.1 \cdot n} \mid n \in \mathbb{N}, 0 \le n \le 80\}$ [S1].
- Optimization steps: 100 [S1].

**Steps:**
1. Initialize the mesh as a sphere [S1].
2. For each of the 24 azimuths, render the current mesh using a hard renderer to obtain target silhouette masks [S1].
3. Define a loss function measuring the difference between the rendered silhouette of the current mesh and the target silhouette [S1].
4. Perform a grid search over the specified learning rates and temperatures for each renderer instance to find optimal hyperparameters [S1].
5. Optimize the mesh vertices using the Adam optimizer for 100 steps, minimizing the silhouette loss [S1].
6. Evaluate the result from five elevation angles $\{-60^\circ, -30^\circ, 0^\circ, 30^\circ, 60^\circ\}$ as independent runs and average the results [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def silhouette_optimization(target_silhouettes_24_azimuths, initial_sphere_mesh, renderer_configs):
    # Initialize mesh vertices as differentiable parameters
    vertices = initial_sphere_mesh.vertices.clone().requires_grad_(True)
    
    # Grid search over hyperparameters
    best_loss = float('inf')
    best_vertices = None
    
    for lr in [10**-1.25, 10**-1.5, 10**-1.75]:
        for n in range(81):
            tau = 10**(-0.1 * n)
            for config in renderer_configs:
                # Create optimizer with specific learning rate
                optimizer = Adam([vertices], lr=lr, betas=(0.5, 0.95))
                
                # Optimization loop
                for step in range(100):
                    optimizer.zero_grad()
                    total_loss = 0
                    
                    # Render from 24 azimuths and compute loss
                    for azimuth in range(24):
                        rendered_silhouette = differentiable_render(vertices, azimuth, config, tau)
                        target = target_silhouettes_24_azimuths[azimuth]
                        loss = silhouette_loss(rendered_silhouette, target)
                        total_loss += loss
                    
                    total_loss.backward()
                    optimizer.step()
                
                # Evaluate from 5 elevation angles
                eval_losses = []
                for elev in [-60, -30, 0, 30, 60]:
                    rendered = differentiable_render(vertices, elev, config, tau)
                    # Note: Target silhouettes for evaluation angles must be available or generated
                    # The source implies averaging results, likely comparing against ground truth silhouettes
                    # or using the same loss metric.
                    eval_loss = silhouette_loss(rendered, target_silhouettes_for_elev[elev])
                    eval_losses.append(eval_loss)
                
                avg_eval_loss = sum(eval_losses) / 5
                
                if avg_eval_loss < best_loss:
                    best_loss = avg_eval_loss
                    best_vertices = vertices.clone().detach()
    
    return best_vertices
```

**Edge Cases and Pitfalls:**
- Limiting to 100 steps is critical to test gradient quality, as longer runs can solve the task with any differentiable renderer [S1].
- The evaluation is computationally expensive, involving 1,242 renderer instances (distributions $\times$ T-conorms) [S1].
- Extreme values for the scale hyperparameter were never optimal, requiring a fine-grained grid search [S1].

## Subtopics

### Edge Splitting
An adaptive mechanism to maintain high-frequency detail by dynamically adjusting the triangle set. It subdivides edges whose view-space length exceeds 1/5 of the image height to encourage uniform view-space edge lengths [S2, unofficial]. This is performed every 100 iterations. View-space edge lengths are computed on a randomly sampled subset of 20 training views to reduce overhead [S2, unofficial]. It is used in conjunction with triangle pruning to enforce a target primitive count [S2, unofficial].

## Operating characteristics
- **Time Complexity:** High due to the need for gradient computation through the renderer for each optimization step. The evaluation involves 1,242 renderer instances with grid searches, making it computationally expensive [S1].
- **Memory Complexity:** Depends on the mesh size and the number of training views stored for edge splitting evaluation (20 views) [S2, unofficial].
- **Strengths:** Effective for shape reconstruction from silhouettes; allows for fine-grained control via hyperparameter search [S1].
- **Limitations:** Computationally expensive; sensitive to hyperparameter choices (learning rate, temperature) [S1].
- **When to use:** When explicit mesh geometry is required and silhouette matching is the primary goal [S1].
- **When not to use:** When computational resources are limited or when voxel-based representations are acceptable [S1].

## Background
Originated in the GenDR paper (2022) as a shape optimization experiment to evaluate differentiable renderers [S1]. The method was used to compare the quality of gradients produced by various distributions and T-conorms in a differentiable rendering context [S1]. The edge splitting variant was introduced in DiffSoup (2026) for extreme radiance field simplification [S2, unofficial].

## Sources
[S1] GenDR: A Generalized Differentiable Renderer, https://doi.org/10.1109/cvpr52688.2022.00397, fetched 2026-10-26
[S2] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
