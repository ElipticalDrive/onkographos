# Shape Deformation for Bandsaw Cutting

Also called: Bandsaw Cutting Deformation

## Overview
A method to deform a triangle mesh to make it more 'cutable' by orthogonal cuts while preserving its overall shape [S1]. It uses rasterized filled silhouettes generated via the mellowmax operator to define a differentiable cut energy, which is minimized against an ARAP regularization term [S1]. It sets itself apart from naive orthogonal cutting by allowing small mesh deformations to remove material that would otherwise obstruct the cut path, thereby reducing the volume difference between the desired cut and the actual result [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A triangle mesh defined by vertices $V$ and face indices $F$.
*   **Output:** A deformed triangle mesh with updated vertex positions.

**Data Structures**
*   **Voxel Grid ($W$):** A 3D grid of size $n \times n \times n$ containing winding numbers. Self-intersections result in winding numbers greater than one [S1].
*   **Silhouette Grids ($S_x, S_y, S_z$):** 2D grids of size $n \times n$ representing rasterized filled silhouettes along the x, y, and z axes [S1].
*   **Cut Shape Grid ($S$):** A 3D grid of size $n \times n \times n$ representing the voxelized shape after orthogonal cutting [S1].

**Parameters**
*   **Voxelization Resolution ($n$):** $128$ [S1]. For a $10\text{cm}^3$ wooden block, this yields a resolution of $< 0.8\text{mm}^3$ [S1].
*   **Mellowmax Temperature ($\beta$):** $1000$ [S1].
*   **Regularization Weight ($\alpha$):** $0.05$ [S1].
*   **Optimizer:** Adam [Kingma and Ba 2017] [S1].
*   **Learning Rate:** $10^{-3}$ [S1].

**Steps**
1.  **Voxelization:** Compute the voxelized shape $W \in \mathbb{R}^{n \times n \times n}$ from the input mesh $V, F$. The values in $W$ represent winding numbers [S1].
2.  **Silhouette Generation:** Generate rasterized filled silhouettes $S_x, S_y, S_z \in \mathbb{R}^{n \times n}$ using the mellowmax operator [S1]. For the x-direction silhouette:
    $$ (S_x)_{ij} = \frac{1}{\beta} \ln\left(\frac{1}{\beta} \sum_{k} \exp(\beta W_{ijk})\right) $$
    The gradients of the mellowmax operator are computed using the softmax function [S1]. If multiple voxels have the same value, they contribute equally to the gradient [S1]. Similar formulas apply for $S_y$ and $S_z$ by permuting indices appropriately.
3.  **Cut Shape Computation:** Compute the voxelized cut shape $S \in \mathbb{R}^{n \times n \times n}$ as the product of the extruded silhouettes:
    $$ S_{ijk} = (S_x)_{ij} (S_y)_{ik} (S_z)_{jk} $$
    This represents the shape resulting from orthogonal cuts along the three axes [S1].
4.  **Energy Calculation:**
    *   Compute the cut energy $E_{\text{cut}}$, which measures the squared voxelized volume differences between the cut shape and the input voxelized shape:
        $$ E_{\text{cut}} = \sum_{ijk} (S_{ijk} - W_{ijk})^2 $$
        Note that $W$ contains winding numbers $>1$ for self-intersections, which intentionally penalizes self-intersections requiring additional volume [S1].
    *   Compute the ARAP energy $E_{\text{arap}}$ for the deformed mesh [S1].
    *   Compute the total energy:
        $$ E = E_{\text{cut}} + \alpha E_{\text{arap}} $$
5.  **Optimization:** Minimize the total energy $E$ with respect to the vertex positions using gradient descent with the Adam optimizer [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def mellowmax(values, beta):
    """
    Computes the mellowmax operator.
    values: numpy array of shape (..., k)
    beta: temperature parameter
    Returns: numpy array of shape (..., 1)
    """
    # Softplus-like operation for max approximation
    # Note: The source formula is 1/beta * ln(1/beta * sum(exp(beta * val)))
    # This specific scaling is unusual for standard mellowmax but follows the source exactly.
    exp_vals = np.exp(beta * values)
    sum_exp = np.sum(exp_vals, axis=-1, keepdims=True)
    result = (1.0 / beta) * np.log((1.0 / beta) * sum_exp)
    return result

def compute_silhouette(W, axis, beta):
    """
    Computes a 2D silhouette from a 3D voxel grid W.
    W: numpy array of shape (n, n, n)
    axis: 0 for x, 1 for y, 2 for z
    beta: temperature parameter
    """
    n = W.shape[0]
    # Permute axes so the silhouette plane is (n, n) and the summation axis is last
    if axis == 0:
        # Sum over k (z-axis) for each (i, j)
        # W[i, j, k] -> sum over k
        # Transpose to (n, n, n) where last dim is k
        W_perm = np.transpose(W, (0, 1, 2)) 
        # Actually, W is (n,n,n). For S_x[i,j], we sum over k.
        # So we sum over axis 2.
        return mellowmax(W, beta) # This sums over the last axis by default in mellowmax if not specified? 
                                  # Let's be explicit.
    
    # Explicit implementation for clarity
    if axis == 0:
        # Sum over k (last axis)
        # W shape (n, n, n). We want result (n, n).
        # Reshape to (n*n, n) for mellowmax? Or just sum over axis 2.
        # The formula is sum_k exp(beta * W_ijk).
        # So we sum over the last dimension.
        exp_W = np.exp(beta * W)
        sum_exp = np.sum(exp_W, axis=2)
        return (1.0 / beta) * np.log((1.0 / beta) * sum_exp)
    elif axis == 1:
        # Sum over j (middle axis) for each (i, k)
        # Result shape (n, n) corresponding to (i, k)
        exp_W = np.exp(beta * W)
        sum_exp = np.sum(exp_W, axis=1)
        # Result is (n, n) for (i, k)
        return (1.0 / beta) * np.log((1.0 / beta) * sum_exp)
    elif axis == 2:
        # Sum over i (first axis) for each (j, k)
        # Result shape (n, n) corresponding to (j, k)
        exp_W = np.exp(beta * W)
        sum_exp = np.sum(exp_W, axis=0)
        # Result is (n, n) for (j, k)
        return (1.0 / beta) * np.log((1.0 / beta) * sum_exp)

def bandsaw_deformation(V, F, n=128, beta=1000, alpha=0.05, lr=1e-3, iterations=1000):
    """
    Deforms mesh V, F to improve cutability.
    """
    # 1. Voxelization
    # [G] Voxelization implementation depends on the specific library or method used to generate W.
    # Assuming a function voxelize_mesh(V, F, n) exists that returns W with winding numbers.
    W = voxelize_mesh(V, F, n)
    
    # 2. Silhouette Generation
    S_x = compute_silhouette(W, axis=0, beta=beta)
    S_y = compute_silhouette(W, axis=1, beta=beta)
    S_z = compute_silhouette(W, axis=2, beta=beta)
    
    # 3. Cut Shape Computation
    # S_ijk = (S_x)_ij * (S_y)_ik * (S_z)_jk
    # S_x is (n, n) -> (i, j)
    # S_y is (n, n) -> (i, k)
    # S_z is (n, n) -> (j, k)
    # We need to broadcast these to (n, n, n)
    
    # Reshape S_x to (n, n, 1) for broadcasting over k
    S_x_3d = S_x[:, :, np.newaxis]
    # Reshape S_y to (n, 1, n) for broadcasting over j
    S_y_3d = S_y[:, np.newaxis, :]
    # Reshape S_z to (1, n, n) for broadcasting over i
    S_z_3d = S_z[np.newaxis, :, :]
    
    S = S_x_3d * S_y_3d * S_z_3d
    
    # 4. Energy Calculation
    # E_cut = sum(S - W)^2
    E_cut = np.sum((S - W) ** 2)
    
    # E_arap = ARAP energy of deformed mesh
    # [G] ARAP energy calculation requires the current vertex positions.
    # Since we are optimizing V, we need to track V.
    # This pseudocode assumes a simplified loop where V is updated.
    
    # 5. Optimization Loop
    # [G] The source mentions minimizing with Adam. A full implementation requires
    # computing gradients of E_cut w.r.t V.
    # The gradient of mellowmax is softmax.
    # The gradient of S w.r.t W involves the chain rule through silhouettes.
    # The gradient of W w.r.t V involves the voxelization gradient.
    
    # Since the source does not provide the explicit gradient descent loop code,
    # we describe the process:
    # Initialize V_current = V
    # For each iteration:
    #   Recompute W from V_current
    #   Recompute S_x, S_y, S_z
    #   Recompute S
    #   Compute E_cut
    #   Compute E_arap
    #   Compute E = E_cut + alpha * E_arap
    #   Compute gradient dE/dV
    #   Update V_current using Adam optimizer
    
    # Note: The source does not provide the explicit update rule or loop structure
    # beyond "minimize the energy with alpha = 0.05 by gradient descent using the Adam method".
    # The implementation of the gradient flow through the voxelization and mellowmax
    # is the key differentiable component.
    
    return V_current
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** For a $10\text{cm}^3$ wooden block, the resolution of $n=128$ gives a voxel size of $< 0.8\text{mm}^3$, which is below the precision of many standard bandsaws [S1].
*   **Strengths:** Preserves the overall meaning of the shapes while substantially improving the cut object compared to naive cutting strategies [S1]. The 'cutability' is demonstrated by physically cutting various examples out of wood and polystyrene foam [S1].
*   **Limitations:** The method relies on voxelization, which introduces discretization artifacts. The resolution is limited to $n=128$ in the source [S1].
*   **When to use:** When a mesh needs to be cut by orthogonal bandsaw cuts and the initial shape is not easily cutable without removing significant material or causing self-intersections.
*   **When not to use:** When high-precision cutting is required beyond the voxel resolution, or when the mesh cannot be reasonably approximated by orthogonal cuts.

## Background
The method was introduced in "Differentiable Voxelization of Surface Representations" [S1]. It addresses the problem of directly using silhouettes for sawing, which often leads to shapes that strongly differ from their initial shape due to material that is not removed but does not belong to the input shape [S1]. The idea is to find small deformations of the input that make the shape more cutable but also preserve its overall shape [S1].

## Sources
[S1] Differentiable Voxelization of Surface Representations, https://doi.org/10.1145/3799902.3811203, fetched 2026-10-27
