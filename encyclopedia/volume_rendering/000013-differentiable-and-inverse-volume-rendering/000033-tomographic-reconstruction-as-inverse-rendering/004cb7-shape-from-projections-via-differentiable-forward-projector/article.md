# Shape from Projections via Differentiable Forward Projector

Also called: CTDR, Differentiable Forward Projector for CT

## Overview
CTDR estimates 3D shapes represented by a triangular mesh directly from tomographic projection data by treating reconstruction as a differentiable rendering process [S1, unofficial]. It distinguishes itself from voxel-based methods by optimizing mesh vertices directly, allowing for explicit topological constraints and robust handling of missing data via collision rollback and Student-t loss functions [S1, unofficial].

## How it works
The method formulates tomographic reconstruction as an optimization problem where the objective is to minimize the discrepancy between measured sinograms and projections rendered from a current mesh estimate.

**Inputs and Outputs:**
- **Input:** 2D tomographic projection images (sinograms) and their corresponding acquisition geometries (source-detector distances, angles).
- **Output:** A 3D triangular mesh (vertices and faces) that best explains the input projections.

**Data Structures:**
- **Mesh:** Defined by a vertex tensor $V \in \mathbb{R}^{N_v \times 3}$ and a face tensor $F \in \mathbb{R}^{N_f \times 3}$ (indices into $V$).
- **Projection Geometry:** Parameters defining the X-ray source position, detector plane orientation, and pixel grid dimensions.

**Algorithm Steps:**
1. **Initialization:** Initialize the mesh $V, F$ (e.g., from a prior shape or random perturbation).
2. **Forward Projection (Differentiable Rasterization):**
   - For each projection angle, render the mesh onto the 2D detector plane.
   - This involves projecting 3D vertices to 2D, rasterizing triangles to compute pixel coverage, and accumulating depth-weighted contributions to simulate X-ray attenuation [S1, unofficial].
   - The output is a synthetic sinogram $\hat{p}$.
3. **Loss Computation:**
   - Compute the loss $\mathcal{L}$ between the measured projection $p$ and the rendered projection $\hat{p}$.
   - Use a robust loss function (e.g., Student-t) to handle outliers and missing data in projection masks [S1, unofficial].
   - Optionally add regularization terms for edge, Laplacian, and flatness to preserve mesh quality [S1, unofficial].
4. **Gradient Computation:**
   - Compute the gradient of the loss with respect to the mesh vertices $\frac{\partial \mathcal{L}}{\partial V}$ using backpropagation through the differentiable rasterizer [S1, unofficial].
5. **Optimization Update:**
   - Update vertex positions $V$ using an optimizer (e.g., Adam) [S1, unofficial].
   - **Collision Rollback:** If the optimization step causes mesh self-intersection or inversion, rollback the step to maintain valid topology [S1, unofficial].
6. **Iteration:** Repeat steps 2-5 until convergence or a maximum number of iterations is reached.

**Formulas:**
The total loss is typically:
$$
\mathcal{L}_{total} = \mathcal{L}_{data}(\hat{p}, p) + \lambda_1 \mathcal{R}_{edge} + \lambda_2 \mathcal{R}_{laplacian} + \lambda_3 \mathcal{R}_{flatness}
$$
where $\mathcal{L}_{data}$ is the projection error, and $\mathcal{R}$ terms are regularizers [S1, unofficial].

**Parameters:**
- Learning rate: Not given in the sources.
- Regularization weights ($\lambda_1, \lambda_2, \lambda_3$): Not given in the sources.
- Optimizer: Adam [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Self-Intersection:** Meshes may self-intersect during optimization, leading to invalid geometry. Handled by collision rollback [S1, unofficial].
- **Missing Data:** Projections may have missing regions (e.g., due to occlusion or detector gaps). Handled by robust loss functions like Student-t [S1, unofficial].
- **Precision:** Floating-point precision in rasterization may affect gradient accuracy [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_mesh(vertices, faces, projections, geometries, num_iterations):
    # Initialize optimizer
    optimizer = Adam([vertices], lr=0.01)
    
    for i in range(num_iterations):
        optimizer.zero_grad()
        
        total_loss = 0.0
        
        for proj, geo in zip(projections, geometries):
            # Differentiable forward projection
            rendered_proj = differentiable_rasterize(vertices, faces, geo)
            
            # Compute data fidelity loss (e.g., Student-t loss)
            data_loss = student_t_loss(rendered_proj, proj)
            
            # Compute regularization losses
            edge_loss = compute_edge_regularization(vertices, faces)
            laplacian_loss = compute_laplacian_regularization(vertices)
            flatness_loss = compute_flatness_regularization(vertices, faces)
            
            # Combine losses
            loss = data_loss + 0.1 * edge_loss + 0.01 * laplacian_loss + 0.001 * flatness_loss
            total_loss += loss
        
        # Backpropagation
        total_loss.backward()
        
        # Optimization step
        optimizer.step()
        
        # Collision rollback check
        if has_self_intersection(vertices, faces):
            rollback_step(vertices, faces)
            
    return vertices, faces
```

## Subtopics

### [CTDR Optimization Loop with Regularization](13348-ctdr-optimization-loop-with-regularization.md)
An iterative optimization process that minimizes a composite loss function—combining data fidelity, edge, Laplacian, and flatness regularization terms—to reconstruct a 3D triangular mesh directly from 2D tomographic projection data. It distinguishes itself from standard differentiable rendering by explicitly enforcing topological constraints via collision rollback and utilizing robust loss functions (Student-t) to handle missing data or outliers in projection masks.

## Operating characteristics
- **Time Complexity:** Depends on the number of iterations, mesh complexity ($N_v, N_f$), and number of projections. Each iteration involves rasterization and backpropagation.
- **Memory Complexity:** Proportional to the mesh size and the number of projections stored in memory.
- **Strengths:** Direct mesh output, explicit topology control, robust to missing data.
- **Limitations:** Requires differentiable rasterization, sensitive to initialization, may get stuck in local minima.
- **When to Use:** When a mesh representation is desired, and topological constraints are important.
- **When Not to Use:** When voxel-based representation is sufficient or when computational resources are limited for iterative optimization.

## Background
The method was introduced in the paper "Shape from Projections via Differentiable Forward Projector for Computed Tomography" published in Ultramicroscopy [S1, unofficial]. It extends differentiable rasterization techniques to the domain of computed tomography, enabling end-to-end optimization of 3D shapes from 2D projections.

## Sources
[S1] jakeoung/ShapeFromProjections, https://github.com/jakeoung/ShapeFromProjections, fetched 2026-10-26 (unofficial)
