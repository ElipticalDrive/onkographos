# SDF Optimization via Finite Differences

Also called: Finite Difference SDF Optimization, Mesh-Difference SDF Training

## Overview
This method optimizes the values of a Signed Distance Field (SDF) and associated learnable weights to minimize the geometric difference between an extracted mesh and a target reference mesh [S1, unofficial]. It differs from standard SDF evaluation by treating SDF values as learnable parameters updated via gradient descent, using finite difference gradients for the SDF rather than analytical derivatives [S1, unofficial]. It builds on [SDF Evaluation Implementation](1658-sdf-evaluation-implementation.md).

## How it works
**Inputs:**
- Target mesh geometry (reference).
- Initial SDF grid values.
- Initial learnable weights: $\beta$ (size $N \times 12$), $\alpha$ (size $N \times 8$), $\gamma$ (size $N$), where $N$ is the number of cubes [S1, unofficial].

**Outputs:**
- Optimized SDF values.
- Optimized weights $\beta, \alpha, \gamma$.

**Data Structures:**
- `sdf`: Vector of SDF values, size $N_{sdf}$ [S1, unofficial].
- `beta`: Matrix, size $N \times 12$, RowMajor [S1, unofficial].
- `alpha`: Matrix, size $N \times 8$, RowMajor [S1, unofficial].
- `gamma`: Vector, size $N$ [S1, unofficial].

**Parameters:**
- `eps`: Finite difference step size, $10^{-4}$ [S1, unofficial].
- `clamp_min`: Lower bound for SDF values, $-1.0$ [S1, unofficial].
- `clamp_max`: Upper bound for SDF values, $1.0$ [S1, unofficial].
- `max_update_count`: Number of SDF values to update per iteration, $\min(100, N_{sdf})$ [S1, unofficial].

**Steps:**
1. **Initialization:** Set $\beta, \alpha, \gamma$ to zero. Add noise: $\beta_{i,j} \leftarrow \beta_{i,j} + \text{noise}() \times 0.1$, $\alpha_{i,j} \leftarrow \alpha_{i,j} + \text{noise}() \times 0.1$, $\gamma_i \leftarrow \gamma_i + \text{noise}() \times 0.1$ for all $i, j$ [S1, unofficial].
2. **Gradient Computation:** Loop $k$ from $0$ to `max_update_count` - 1:
   a. Select index $i$ (e.g., sequentially or randomly) [S1, unofficial].
   b. Store `sdf[i]` as `sdf_orig`.
   c. Perturb: `sdf[i] += eps`.
   d. Extract mesh `mesh_plus` using current SDF and weights [S1, unofficial].
   e. Compute loss `loss_plus` between `mesh_plus` and target [S1, unofficial].
   f. Perturb: `sdf[i] -= 2 * eps` (resulting in `sdf_orig - eps`).
   g. Extract mesh `mesh_minus` [S1, unofficial].
   h. Compute loss `loss_minus` [S1, unofficial].
   i. Reset: `sdf[i] = sdf_orig` [S1, unofficial].
   j. Compute gradient: `sdf_grad[i] = (loss_plus - loss_minus) / (2 * eps)` [S1, unofficial].
3. **Optimization Step:** Apply optimizer step: `optimizer.step(sdf, sdf_grad)` [S1, unofficial].
4. **Clamping:** Clamp all SDF values to $[-1.0, 1.0]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def optimize_sdf_finite_difference(sdf, beta, alpha, gamma, target_mesh, eps=1e-4, clamp_min=-1.0, clamp_max=1.0, max_updates=100):
    """
    Optimizes SDF values and weights using finite difference gradients.
    Note: This pseudocode simplifies the weight update mechanism described in [S1, unofficial].
    The source indicates beta, alpha, gamma are learnable and initialized with noise,
    but the gradient step description focuses on sdf_grad. We assume a standard optimizer
    handles all learnable parameters (sdf, beta, alpha, gamma) together, or that
    sdf_grad is part of a larger gradient vector.
    """
    N_sdf = len(sdf)
    N_cubes = beta.shape[0]
    
    # Initialization
    beta[:] = 0.0
    alpha[:] = 0.0
    gamma[:] = 0.0
    
    # Add noise
    beta += np.random.randn(*beta.shape) * 0.1
    alpha += np.random.randn(*alpha.shape) * 0.1
    gamma += np.random.randn(*gamma.shape) * 0.1
    
    # Optimizer state (placeholder for actual optimizer logic)
    # In practice, this would be an Adam or SGD instance initialized with sdf, beta, alpha, gamma
    
    for iteration in range(num_iterations): # num_iterations not specified in sources [G]
        # Gradient computation for a subset of SDF values
        sdf_grad = np.zeros_like(sdf)
        
        # Select indices to update
        indices = np.random.choice(N_sdf, size=min(max_updates, N_sdf), replace=False)
        
        for i in indices:
            sdf_orig = sdf[i]
            
            # Forward perturbation +
            sdf[i] = sdf_orig + eps
            mesh_plus = extract_mesh(sdf, beta, alpha, gamma) # Implementation detail from [S1, unofficial]
            loss_plus = compute_mesh_difference(mesh_plus, target_mesh)
            
            # Forward perturbation -
            sdf[i] = sdf_orig - eps
            mesh_minus = extract_mesh(sdf, beta, alpha, gamma)
            loss_minus = compute_mesh_difference(mesh_minus, target_mesh)
            
            # Reset
            sdf[i] = sdf_orig
            
            # Central difference gradient
            sdf_grad[i] = (loss_plus - loss_minus) / (2 * eps)
            
        # Apply gradient step
        # The source says "optimizer.step(sdf, sdf_grad)". 
        # We assume the optimizer also handles gradients for beta, alpha, gamma if they are learnable.
        # Since the source only details sdf_grad calculation, we apply it to sdf.
        # [G] Assuming a simple gradient descent step for clarity, though Adam is common.
        sdf -= learning_rate * sdf_grad
        
        # Clamp SDF values
        sdf = np.clip(sdf, clamp_min, clamp_max)
        
    return sdf, beta, alpha, gamma

def extract_mesh(sdf, beta, alpha, gamma):
    """
    Extracts a mesh from the SDF using the FlexiCubes method or similar.
    This is a placeholder for the actual mesh extraction algorithm.
    """
    # Implementation depends on the specific SDF primitive definition and extraction method.
    # Based on [S1, unofficial], this likely involves FlexiCubes.
    pass

def compute_mesh_difference(mesh1, mesh2):
    """
    Computes a loss metric between two meshes (e.g., Chamfer distance, L2 distance).
    """
    pass
```

**Edge Cases and Pitfalls:**
- Precision: The finite difference step `eps` must be small enough to approximate the derivative but large enough to avoid numerical noise in mesh extraction and loss computation [S1, unofficial].
- Boundaries: SDF values are clamped to $[-1.0, 1.0]$, which may affect gradients near the boundaries [S1, unofficial].
- Degenerate Input: If the target mesh is empty or invalid, the loss computation may fail [G].

## Subtopics

## Operating characteristics
- **Time Complexity:** Each gradient computation for one SDF value requires two mesh extractions and two loss computations. With `max_updates` = 100, this is $O(100 \cdot (M_{extract} + L_{compute}))$ per iteration, where $M_{extract}$ is mesh extraction cost and $L_{compute}$ is loss computation cost.
- **Memory Complexity:** Stores SDF, beta, alpha, gamma, and temporary meshes for perturbation [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Updating only 100 SDF values per iteration allows for faster convergence or larger grids, but may slow down global convergence [S1, unofficial].
- **Strengths:** Does not require analytical gradients for the SDF or mesh extraction, making it applicable to complex implicit surface definitions [S1, unofficial].
- **Limitations:** Computationally expensive due to repeated mesh extractions. Gradient noise may affect stability [G].
- **When to use:** When analytical gradients are unavailable or too complex to derive, and the SDF representation is flexible enough to be optimized directly [S1, unofficial].
- **When not to use:** When real-time performance is critical, or when the SDF is defined by simple primitives with known analytical gradients.

## Background
This method is associated with the FlexiCubes framework, which uses a learnable SDF representation for mesh optimization [S1, unofficial]. The approach of optimizing SDF values directly via finite differences is a technique used in neural implicit surface reconstruction and mesh optimization pipelines.

## Sources
[S1] ddiakopoulos/flexicubes_cpp, https://github.com/ddiakopoulos/flexicubes_cpp, fetched 2026-10-27 (unofficial)
