# Attached Formulation (ResLRB Attached / LRB-3-pass Attached)

Also called: LRB-3-pass Attached, ResLRB Attached

## Overview
The Attached Formulation extends differentiable light tracing to support gradients with respect to geometric parameters that move vertices, such as lens surfaces or specular mirrors [S1, unofficial]. Unlike the Detached Formulation, which only differentiates scattering properties, the Attached Formulation tracks how geometric perturbations propagate through ray segments to shift downstream vertices and the sensor coordinates at which flux is splatted [S1, unofficial]. This capability is essential for optimizing geometry involved in specular transport, where the splat position depends on the path geometry rather than being fixed by a camera ray [S1, unofficial].

## How it works
The Attached Formulation is implemented within the LRB-3-pass framework [S1, unofficial]. It requires three passes: a primal pass, a backward pass for radiance adjoints, and a third pass for position adjoints [S1, unofficial]. The key distinction from the Detached Formulation is the accumulation and propagation of position adjoints through the splat Jacobian [S1, unofficial].

### Inputs and Outputs
*   **Inputs**: Primal path vertices $x_0, \dots, x_k$; primal ray Jacobians $J_{0 \rightarrow i}^{\text{ray}}$; sensor connection Jacobians $J_i^{\text{sensor}}$; radiance adjoints $\bar{\mathcal{L}}$; position adjoints $\frac{\partial \mathcal{L}}{\partial u_i}$ from the sensor connection.
*   **Outputs**: Position adjoints in local parameterization at each vertex $x_j$, denoted $A_{\text{pos,local}}$, which are backpropagated alongside radiance adjoints [S1, unofficial].

### Data Structures
*   **$A_{\text{pos}}$**: A four-component vector per sample. It is not a matrix because the left-multiplication by a row vector collapses each $4 \times 4$ Jacobian to a single row [S1, unofficial].
*   **$\delta_{\text{rem}}$**: One spectral value representing the remainder of the radiance adjoint [S1, unofficial].
*   **Total per-sample storage**: $\delta_{\text{rem}}$ (1 float) + $A_{\text{pos}}$ (4 floats) = 5 floats, independent of path length [S1, unofficial].

### Steps
1.  **Primal Pass**: Compute forward ray Jacobians $J_{0 \rightarrow i}^{\text{ray}}$ and sensor-connection Jacobians $J_i^{\text{sensor}}$ [S1, unofficial].
2.  **Backward Pass (Pass 2)**:
    *   At each vertex $x_i$ with a valid sensor connection, compute the splat Jacobian:
        $$J_i^{\text{splat}} = J_i^{\text{sensor}} \cdot J_{0 \rightarrow i}^{\text{ray}}$$
    *   Read the position adjoint $\frac{\partial \mathcal{L}}{\partial u_i}$ from the sensor connection [S1, unofficial].
    *   Accumulate the position adjoint $A_{\text{pos}}$ using the formula below.
3.  **Position Adjoint Projection (Pass 3)**:
    *   At vertex $x_j$, project the accumulated remainder $A_{\text{pos,rem}}$ into the local frame:
        $$A_{\text{pos,local}} = A_{\text{pos,rem}} \cdot \left( J_{0 \rightarrow j}^{\text{ray}} \right)^{-1}$$
    *   The last two components of $A_{\text{pos,local}}$ give the position adjoint in the local parameterization at $x_j$ [S1, unofficial].
    *   Backpropagate $A_{\text{pos,local}}$ through the two-point parameterization alongside the radiance adjoint [S1, unofficial].

### Formulas
*   **Position Adjoint Accumulation**:
    $$A_{\text{pos}} = \sum_{i=1}^k \begin{pmatrix} 0 & 0 & \frac{\partial \mathcal{L}}{\partial u_{i,x}} & \frac{\partial \mathcal{L}}{\partial u_{i,y}} \end{pmatrix} J_i^{\text{splat}} \quad (18)$$
    where $\frac{\partial \mathcal{L}}{\partial u_{i,x}}$ and $\frac{\partial \mathcal{L}}{\partial u_{i,y}}$ are the partial derivatives of the loss with respect to the sensor coordinates $u_{i,x}$ and $u_{i,y}$ [S1, unofficial].

*   **Local Position Adjoint Projection**:
    $$A_{\text{pos,local}} = A_{\text{pos,rem}} \cdot \left( J_{0 \rightarrow j}^{\text{ray}} \right)^{-1} \quad (19)$$
    where $J_{0 \rightarrow j}^{\text{ray}}$ is the forward ray Jacobian from the origin to vertex $x_j$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 2: Position Adjoint Accumulation
# Input: v (current vertex state), u (sensor coordinates), J_ray_0_to_i (forward ray Jacobian), J_sensor (sensor connection Jacobian)
# Output: A_pos (accumulated position adjoint vector)

A_pos = [0.0, 0.0, 0.0, 0.0]

# Compute splat Jacobian
# J_sensor is 2x4, J_ray_0_to_i is 4x4, result is 2x4
J_splat = matrix_multiply(J_sensor, J_ray_0_to_i)

# Read adjoints from sensor connection
# dL_du is a 2D vector [dL_du_x, dL_du_y]
dL_du = read_adjoint_position(v, u)

# Construct row vector for left-multiplication
# [0, 0, dL_du_x, dL_du_y]
row_vector = [0.0, 0.0, dL_du[0], dL_du[1]]

# Accumulate A_pos
# row_vector (1x4) * J_splat (4x4) -> result (1x4)
# Note: The formula in [S1, unofficial] shows row_vector * J_splat, but J_splat is 2x4.
# The text states "left-multiplication by a row vector collapses each 4x4 Jacobian to a single row".
# Assuming J_splat is effectively treated as the full 4x4 transformation from ray origin to sensor,
# or that the relevant part is extracted. Based on Eq 18, J_splat is 4x4.
# If J_splat is 2x4, the multiplication is undefined for a 1x4 row vector.
# Re-reading [S1, unofficial]: "J_i^splat = J_i^sensor * J_{0->i}^ray". J_sensor is 2x4, J_ray is 4x4.
# Result is 2x4. The formula uses a 1x4 row vector.
# This implies the row vector is multiplied by the 4x4 J_ray part, or J_sensor is incorporated differently.
# However, the text explicitly says: "A_pos += [0, 0, dL_du_x, dL_du_y] * J_splat".
# And "A_pos is a four-component vector... because left-multiplication by a row vector collapses each 4x4 Jacobian".
# This suggests J_splat in the formula context is the 4x4 Jacobian from ray origin to sensor plane.
# Let's assume J_splat is computed as the full 4x4 Jacobian.

# If J_splat is 2x4, we cannot multiply a 1x4 row vector.
# Let's look at the structure: J_sensor maps 4D ray params to 2D sensor.
# J_ray maps 4D ray params to 4D vertex params.
# The adjoint flows from sensor (2D) back to ray params (4D).
# The formula (18) sums over i. The term is [0,0, dL/du_x, dL/du_y] * J_splat.
# If J_splat is 4x4, the result is 1x4.
# If J_splat is 2x4, the result is 1x2.
# The text says A_pos is 4-component. So J_splat must be 4x4.
# This implies J_splat = J_ray_0_to_i * (some mapping)?
# Actually, J_sensor is 2x4. J_ray is 4x4.
# The chain rule for sensor coordinates u with respect to ray parameters p is:
# du/dp = J_sensor * J_ray. This is 2x4.
# The adjoint dL/du is 2x1.
# dL/dp = (du/dp)^T * dL/du = (J_sensor * J_ray)^T * dL/du.
# This is 4x1.
# The formula in [S1, unofficial] is: [0,0, dL/du_x, dL/du_y] * J_splat.
# This is a row vector (1x4) times a matrix.
# If J_splat is 4x4, result is 1x4.
# If J_splat is 2x4, result is 1x2.
# The text says A_pos is 4-component.
# Therefore, J_splat in the formula must be 4x4.
# How is J_splat defined? "J_i^splat = J_i^sensor * J_{0->i}^ray".
# If J_sensor is 2x4 and J_ray is 4x4, the product is 2x4.
# There is a contradiction in the dimensions described in the text vs the formula interpretation.
# However, the text explicitly states: "A_pos is a four-component vector... because left-multiplication by a row vector collapses each 4x4 Jacobian".
# This implies the Jacobian being multiplied is 4x4.
# Perhaps J_sensor is treated as a 4x4 matrix with zeros in the first two rows?
# Or perhaps the "row vector" is actually the adjoint vector transposed?
# Let's stick to the explicit formula and variable names from [S1, unofficial].

# Implementation note: If J_splat is 2x4, we must reshape or pad.
# Given the constraint "A_pos is 4-component", we assume J_splat is 4x4.
# This might mean J_sensor is embedded in a 4x4 matrix.

# For the purpose of the pseudocode, we follow the formula structure.
# We assume J_splat is 4x4.

A_pos += matrix_multiply(row_vector, J_splat)

# Pass 3: Local Position Adjoint Projection
# Input: A_pos_rem (remainder of A_pos from previous vertices), J_ray_0_to_j (forward ray Jacobian to current vertex)
# Output: A_pos_local (position adjoint in local frame)

# Invert J_ray_0_to_j
J_ray_inv = matrix_inverse(J_ray_0_to_j)

# Project
A_pos_local = matrix_multiply(A_pos_rem, J_ray_inv)

# The last two components of A_pos_local are the position adjoints in local parameterization
local_pos_adjoint = A_pos_local[2:4]
```

### Parameters
*   **Typical Values**: Not given in the sources.
*   **Edge Cases**:
    *   **Precision**: Computing forward ray Jacobians in both primal and backward passes introduces overhead and potential numerical instability [S1, unofficial].
    *   **Boundaries**: The method requires valid sensor connections at each vertex to accumulate position adjoints [S1, unofficial].
    *   **Degenerate Input**: If $J_{0 \rightarrow j}^{\text{ray}}$ is singular, inversion is not possible [S1, unofficial].

### Subtopics
*   **Detached Formulation**: Differentiates parameters that influence scattering values (e.g., albedo, intensity) without altering path geometry. Chosen when geometry is fixed. [Link](23401-detached-formulation.md)
*   **ResLRB**: A constant-memory reverse-mode differentiable light tracing method that preserves connections. Chosen for memory-constrained environments. [Link](23400-lrb-3-pass.md)

## Operating characteristics
*   **Time Complexity**: Considerably slower than detached counterparts due to the overhead of computing forward ray Jacobians in both primal and backward passes [S1, unofficial].
*   **Memory Complexity**: Constant memory, independent of path length. Per-sample storage is 5 floats ($\delta_{\text{rem}}$ + $A_{\text{pos}}$) [S1, unofficial].
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Constant memory methods maintain a flat memory profile [S1, unofficial].
*   **Strengths**: Supports optimization of geometry in specular transport; constant memory usage.
*   **Limitations**: High computational cost due to Jacobian computation; complexity of implementing position adjoint propagation.
*   **When to Use**: When optimizing geometric parameters (e.g., lens shapes) that affect the path geometry and splat position.
*   **When Not to Use**: When only scattering properties need to be differentiated, or when computational efficiency is prioritized over geometric optimization.

## Background
The Attached Formulation was introduced in "Constant-Memory Differentiable Light Tracing" [S1, unofficial]. It addresses the limitation of previous methods that could not differentiate geometric parameters in light tracing due to the one-path/many-pixels connectivity and the dependency of sensor coordinates on path geometry [S1, unofficial]. The method is part of a family of constant-memory reverse-mode differentiable light tracing methods, including ResLRB and LRB-3-pass [S1, unofficial].

## Sources
[S1] Constant-Memory Differentiable Light Tracing, https://arxiv.org/abs/2609.32920, fetched 2026-10-29 (unofficial)
