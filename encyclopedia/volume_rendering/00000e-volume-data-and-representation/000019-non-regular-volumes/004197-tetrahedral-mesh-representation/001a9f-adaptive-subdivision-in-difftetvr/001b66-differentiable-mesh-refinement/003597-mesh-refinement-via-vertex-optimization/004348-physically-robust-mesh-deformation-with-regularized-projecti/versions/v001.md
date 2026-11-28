# Physically Robust Mesh Deformation with Regularized Projection

Also called: Regularized Projection Optimization

## Overview
This method jointly optimizes the locations of projected vertices in a volumetric tetrahedral mesh to avoid imprecise predictions, non-uniform distortion, and flipping artifacts that arise from naive projection [S1, unofficial]. It sets itself apart from simple closest-point mapping by introducing a differentiable optimization formulation that combines a robust projection data term with smoothness and orientation regularization terms [S1, unofficial]. It builds on [Mesh Refinement via Vertex Optimization](989-mesh-refinement-via-vertex-optimization.md) by providing the specific objective function and regularization details required for the final deformation step.

## How it works
**Inputs and Outputs**
- **Input**: A voxelized shape and an initial tetrahedral mesh $V$; a neural network $h_\theta(V, \cdot)$ that predicts the closest point on the implicit surface for any given vertex [S1, unofficial].
- **Output**: A deformed volumetric mesh with optimized vertex positions $V$ [S1, unofficial].

**Data Structures**
- $V$: Set of vertices in the mesh.
- $V^s$: Subset of vertices on the surface of the mesh.
- $F^s_{tri}$: Set of surface triangles.
- $M_{c,k}$: A $4 \times 4$ matrix stacked with the homogeneous coordinates of the four vertices of the $k$-th tetrahedron in the $c$-th cube [S1, unofficial].
- $n$: Random Gaussian noise vector $n \sim \mathcal{N}(0, 1)$ [S1, unofficial].

**Parameters**
- $k$: Coefficient for robust projection noise, gradually decays to 0 [S1, unofficial].
- $v_0$: Constant thresholding value to activate the orientation penalty, ensuring a minimum volume of tetrahedra [S1, unofficial].
- $\lambda_a, \lambda_b, \lambda_c$: Hyper-parameters for the smooth term, determined via cross-validation [S1, unofficial].
- Iterations: 60 iterations for the main optimization [S1, unofficial]. Note: An alternative schedule uses 80 steps with a specific noise decay [S1, unofficial].

**Formulas**
The total loss function $L(V)$ is defined as [S1, unofficial]:
$$
L(V) = \sum_{v^s \in V^s} \tilde{L}_{proj}(v^s; h_\theta(V, \cdot)) + R_{smooth}(V) + R_{orientation}(V)
$$

1. **Robust Projection Term** $\tilde{L}_{proj}$:
   $$
   \tilde{L}_{proj}(v^s; h_\theta(V, \cdot)) = \| v^s - h_\theta(V, v^s) + k n \|_2
   $$
   where $n \sim \mathcal{N}(0, 1)$ is random Gaussian noise and $k$ is a coefficient that gradually decays to 0 [S1, unofficial].

2. **Smooth Term** $R_{smooth}(V)$:
   $$
   R_{smooth}(V) = \lambda_a R_{edge}(V, F^s_{tri}) + \lambda_b R_{laplacian}(V, F^s_{tri}) + \lambda_c R_{normal}(V^s, F^s_{tri})
   $$
   - $R_{edge}$: Penalizes extremely long edges.
   - $R_{laplacian}$: Graph-based Laplacian term.
   - $R_{normal}$: Enforces normal consistency among adjacent faces on the surface [S1, unofficial].

3. **Orientation Term** $R_{orientation}(V)$:
   $$
   R_{orientation}(V) = \sum_{c,k} l(M_{c,k})
   $$
   where $l(M_{c,k})$ is defined as [S1, unofficial]:
   $$
   l(M_{c,k}) = 
   \begin{cases} 
   -(\det(M_{c,k}) - v_0)^2 \log(\det(M_{c,k})) & \text{if } \det(M_{c,k}) \le v_0 \\
   \det(M_{c,k}) & \text{if } \det(M_{c,k}) > v_0
   \end{cases}
   $$
   The vertices order in $M_{c,k}$ must satisfy the right-hand rule convention such that every tetrahedron in the initial mesh has a positive volume [S1, unofficial].

**Algorithm**
1. Initialize vertex positions $V$ from the initial tetrahedral mesh.
2. For iteration $t = 1$ to $T$ (where $T=60$ or $T=80$ depending on the variant):
   a. Compute the predicted closest points $p^s = h_\theta(V, v^s)$ for all $v^s \in V^s$.
   b. Generate random Gaussian noise $n \sim \mathcal{N}(0, 1)$ for each vertex.
   c. Determine the noise coefficient $k_t$.
      - If using the 60-iteration variant, $k$ decays to 0 [S1, unofficial].
      - If using the 80-step variant: Start with $k=0.1$. Decrease $k$ by 0.5 every 10 steps [S1, unofficial].
   d. Compute the robust projection residual: $r^s = v^s - p^s + k_t n$.
   e. Compute the smooth regularization terms $R_{edge}$, $R_{laplacian}$, and $R_{normal}$ based on current $V$ and $F^s_{tri}$ [S1, unofficial].
   f. Compute the orientation regularization terms by calculating $\det(M_{c,k})$ for all tetrahedra and applying the function $l(\cdot)$ [S1, unofficial].
   g. Compute the total loss $L(V)$ using the formulas above.
   h. Update $V$ using a gradient-based optimizer (e.g., Adam) to minimize $L(V)$ [S1, unofficial].
3. Return the final deformed mesh $V$.

**Edge Cases and Pitfalls**
- Naive projection $v^s \leftarrow h_\theta(V, v^s)$ causes bumps, non-uniform distortion, and flipping artifacts [S1, unofficial].
- The vertices order in $M_{c,k}$ must satisfy the right-hand rule convention such that every tetrahedron in the initial mesh has a positive volume [S1, unofficial].
- The function $l(\cdot)$ is a robust $C^2$-function, which is widely used in physical simulation [S1, unofficial].

## Subtopics

### Regularization Terms Ablation
An ablation study evaluates the contribution of different components in $L(V)$: one-step closest-point mapping, $L_{proj}$, $\tilde{L}_{proj}$, $L_{proj} + R_{smooth}$, $L_{proj} + R_{orientation}$, $L_{proj} + R_{smooth} + R_{orientation}$, and All (surface + smooth + orientation + noise) [S1, unofficial].
- **Chair mesh generation**: 'All' configuration achieved triAR mean=1.45, tetAR mean=2.19, triFlip=0.91, tetFlip=0.00, self-intersection=0.48. 'Lproj + Rsmooth + Rorientation' achieved triAR mean=1.59, tetAR mean=2.28, triFlip=2.82, tetFlip=0.01, self-intersection=2.74 [S1, unofficial].
- **Lamp mesh generation**: 'All' configuration achieved triAR mean=1.78, tetAR mean=2.56, triFlip=0.44, tetFlip=0.00, self-intersection=0.18. 'Lproj + Rsmooth + Rorientation' achieved triAR mean=1.94, tetAR mean=2.68, triFlip=3.43, tetFlip=0.00, self-intersection=1.63 [S1, unofficial].
The smooth term contributes significantly to enhancing every aspect of mesh quality. The orientation term removes all flipped tetrahedra and nearly all flipped triangles. The effective noise/stochastic term helps clean mesh quality further. One-step mapping has better quality than naive multi-step optimization but lacks room for improvement [S1, unofficial].

### Robust Projection with Regularized Projection
This variant specifies a noise schedule for Robust Projection that adds stronger noise at the beginning stage and gradually decreases it at the converge stage [S1, unofficial].
- Optimization runs for 80 steps, which is empirically sufficient for a voxel shape to converge to the volumetric mesh [S1, unofficial].
- Gaussian noise is added with a ratio $k$ in a linear scheduler to suppress distortion. Starting with initial ratio $k=0.1$, $k$ is decreased by 0.5 every 10 steps until 80 steps are completed [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: The method is evaluated on ShapeNet Chair and Airplane categories [S1, unofficial]. Training uses 4x RTX2080Ti GPUs [S1, unofficial].
- **Strengths**: Removes flipped tetrahedra and nearly all flipped triangles; reduces distortion and bumps; avoids collisions between vertices [S1, unofficial].
- **Limitations**: Requires gradient-based optimization; hyper-parameters $\lambda_a, \lambda_b, \lambda_c$ need cross-validation [S1, unofficial].
- **When to use**: When generating high-quality volumetric tetrahedral meshes from voxelized shapes or implicit fields, especially when avoiding topological defects like flips is critical [S1, unofficial].
- **When not to use**: When computational resources for gradient-based optimization are limited, or when a simpler, faster approximation is acceptable despite potential artifacts [S1, unofficial].

## Background
Originated in the Neural Volumetric Mesh Generator (NVMG) paper [S1, unofficial]. The method addresses issues with naive projection in mesh deformation, such as imprecise predictions, non-uniform distortion, and flipping artifacts [S1, unofficial]. It builds on prior work in physical simulation using robust $C^2$-functions for orientation preservation [S1, unofficial].

## Sources
[S1] Neural Volumetric Mesh Generator, https://arxiv.org/abs/2210.03158v1, fetched 2026-10-27 (unofficial)
