# Gradient-Based Mesh Optimization

Also called: nvdiffrec-style optimization

## Overview
Gradient-Based Mesh Optimization reconstructs unknown meshes by optimizing a randomly initialized Signed Distance Field (SDF) to minimize geometric differences against a reference mesh using multiview mask and depth losses [S1, unofficial]. It builds on [SDF Optimization via Finite Differences](3086-sdf-optimization-via-finite-differences.md) by treating the SDF values as learnable parameters updated via gradient descent rather than fixed analytical functions [G].

## How it works
**Inputs:**
- Target mesh (reference geometry) for loss computation [S1, unofficial].
- Multiview masks or depth maps of the target [S1, unofficial].
- Randomly initialized SDF grid or point cloud [S1, unofficial].

**Outputs:**
- Optimized SDF field [S1, unofficial].
- Extracted mesh matching the target [S1, unofficial].

**Data Structures:**
- SDF values: Learnable parameters updated via gradient descent [S1, unofficial]. Exact tensor layout and bit packing are Not given in the sources.

**Steps:**
1. **Initialize SDF:** Randomly initialize the SDF values at the grid points or vertices [S1, unofficial].
2. **Optimize SDF:** Use gradient descent to minimize multiview mask and depth losses against the reference mesh [S1, unofficial].
3. **Regularization (Optional):** Apply a developability regularizer ($L_{dev}$) to encourage fabricability from panels [S1, unofficial].
   - Regularization weights are initially set low to avoid hindering convergence and are incremented if artifacts appear [S1, unofficial].
   - An L1 regularizer on flexible weights may be applied to stabilize generative pipelines (e.g., GET3D), though it has low impact for single shapes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize SDF values randomly
sdf_values = torch.randn(num_points)
sdf_values.requires_grad = True
optimizer = torch.optim.Adam([sdf_values], lr=0.01)
reg_weight = 0.0
flex_reg_weight = 0.0

for iteration in range(num_iterations):
    optimizer.zero_grad()
    # [G] Differentiable forward pass through renderer/extractor to get masks and depths
    rendered_masks = forward_pass_masks(sdf_values)
    rendered_depths = forward_pass_depths(sdf_values)
    # Compute losses against target reference
    mask_loss = torch.nn.functional.binary_cross_entropy(rendered_masks, target_masks)
    depth_loss = torch.nn.functional.l1_loss(rendered_depths, target_depths)
    total_loss = mask_loss + depth_loss
    # Optional regularization
    if use_developability:
        dev_loss = compute_developability_loss(sdf_values)
        total_loss += reg_weight * dev_loss
    if use_flexible_weight_reg:
        flex_reg = torch.norm(sdf_values, p=1)
        total_loss += flex_reg_weight * flex_reg
    total_loss.backward()
    optimizer.step()
    # Increment regularization weights if artifacts appear
    if artifacts_detected:
        reg_weight *= 1.1
        flex_reg_weight *= 1.1
```

**Edge Cases and Pitfalls:**
- **Precision:** SDF values must be precise enough to distinguish inside/outside [G].
- **Boundaries:** Ensure the SDF domain covers the entire target mesh [G].
- **Degenerate Input:** If the target mesh has holes or non-manifold geometry, loss computation may be unstable [G].

## Subtopics

### Gradient-Based Mesh Optimization via Sphere Fitting
Approximates non-deformed initial mesh geometry with a set of spheres to compute visibility [S2]. The mesh is deformed by translating and rotating these spheres, allowing inverse problems to supervise sphere parameters and update geometry [S2].
- **Steps:**
  1. Determine initial position and radius of each sphere by minimizing the total volume occupied by the sphere set outside the mesh (sphere outside volume) along with a term encouraging the sphere set to cover the mesh surface. Optimized using gradient descent [S2].
  2. Connect sphere centers as an embedded graph to drive mesh deformation. Each sphere is connected to its K-nearest vertices on the template [S2].
  3. Parameterize mesh deformation by the rotation and translation of each sphere. Optimize these parameters using rendering loss during inverse problems [S2].
- **When to choose:** When visibility computation is a bottleneck and the geometry can be approximated by spheres [S2].

## Operating characteristics
Not given in the sources.

## Background
Developed as part of the FlexiCubes workflow for mesh reconstruction [S1, unofficial]. It operates within differentiable rendering and inverse graphics pipelines [G].

## Sources
[S1] GitHub - nv-tlabs/FlexiCubes, https://github.com/nv-tlabs/FlexiCubes, fetched 2026-10-27 (unofficial)
[S2] Efficient and Differentiable Shadow Computation for Inverse Problems, https://doi.org/10.1109/iccv48922.2021.01286, fetched 2026-11-01
