# TensoRF Training Configuration

Also called: Anatomy-Regularized TensoRF, AReT Configuration

## Overview
This configuration defines the training protocol for the TensoRF representation in the context of sparse-view lung nodule volumetry from digitally reconstructed radiographs (DRRs) [S1, unofficial]. It addresses the instability of standard TensoRF density parameterization, which causes zero gradient flow during early training, and introduces anatomy-aware regularization to prevent overfitting in highly under-constrained settings (18:1 parameter-to-ray ratio) [S1, unofficial]. It sets itself apart from standard NeRF variants by using vector-matrix (VM) tensor factorization for the density field and specific initialization shifts to ensure non-zero density gradients [S1, unofficial].

## How it works
**Inputs:**
*   Sparse set of DRR images (e.g., 72 views from MedNeRF-rendered data) [S1, unofficial].
*   Camera poses corresponding to the input views [S1, unofficial].
*   Scene bounding box (AABB): $x \in [-2.5, 3.5]$, $y \in [-1.5, 1.5]$, $z \in [-2.0, 1.5]$ [S1, unofficial].

**Outputs:**
*   Optimized density grid parameters and appearance MLP weights [S1, unofficial].
*   A volumetric density field $\sigma(\mathbf{x})$ suitable for Marching Cubes extraction [S1, unofficial].

**Data Structures:**
1.  **Density Grid (VM Decomposition):**
    *   Layout: Vector-Matrix decomposition with rank 16 per axis [S1, unofficial].
    *   Dimensions: $16 \times 16 \times 16$ voxel grid, resulting in $64^3$ voxels [S1, unofficial].
    *   Voxel spacing: Approximately 3.75 mm per voxel (anisotropic: $[3.75, 1.875, 2.1875]$ mm) [S1, unofficial].
    *   Parameters: Density feature planes $P_i$ and lines $L_i$ [S1, unofficial].
2.  **Appearance MLP:**
    *   Hidden layers: 2 layers with 128 units each [S1, unofficial].
    *   Activation: ReLU [S1, unofficial].
    *   Input features: 27-dimensional appearance features [S1, unofficial].
    *   Output: Color $\mathbf{c} \in \mathbb{R}^3$ [S1, unofficial].
3.  **Positional Encoding:**
    *   Frequencies: 6 [S1, unofficial].

**Training Steps:**
1.  **Initialization:**
    *   Initialize density feature planes/lines such that $f_\theta(\mathbf{x}) \in [-1, 1]$ [S1, unofficial].
    *   Set density shift $\Delta = 0$ and scale $s = 1$ [S1, unofficial].
    *   *Note:* The default parameterization ($\Delta = -10, s = 25$) is avoided because it results in $f_\theta(\mathbf{x}) + \Delta \in (-\infty, 0)$, causing the ReLU in the density equation to be identically zero, yielding zero density and zero photometric gradient flow [S1, unofficial].
2.  **Density Computation:**
    *   Compute raw density output $f_\theta(\mathbf{x})$ from the VM decomposition [S1, unofficial].
    *   Apply the corrected density parameterization:
        $$ \sigma(\mathbf{x}) = \text{ReLU}(f_\theta(\mathbf{x}) \cdot s + \Delta) $$
        With $\Delta=0, s=1$, this simplifies to $\sigma(\mathbf{x}) = \text{ReLU}(f_\theta(\mathbf{x}))$ [S1, unofficial].
3.  **Rendering:**
    *   Sample rays (batch size 2048) [S1, unofficial].
    *   Query $\sigma(\mathbf{x})$ and color $\mathbf{c}(\mathbf{x}, \mathbf{d})$ along rays [S1, unofficial].
    *   Integrate using volume rendering equation to produce predicted image $\hat{I}$ [S1, unofficial].
4.  **Loss Calculation:**
    *   Compute photometric loss: $L_{photo} = \text{MSE}(\hat{I}, I)$ [S1, unofficial].
    *   Compute sparsity loss (L1 penalty on density planes):
        $$ L_{sparse} = \sum_i \| P_i \|_1 $$
        Encodes domain knowledge that the thoracic cavity is predominantly air [S1, unofficial].
    *   Compute Total Variation (TV) regularization on density planes:
        $$ L_{TV} = \sum_i \left( \| \Delta_h P_i \|_1 + \| \Delta_v P_i \|_1 \right) $$
        where $\Delta_h, \Delta_v$ are horizontal and vertical finite differences [S1, unofficial].
    *   Total loss:
        $$ L = L_{photo} + \lambda_1 L_{sparse} + \lambda_2 L_{TV} $$
        With $\lambda_1 = 1 \times 10^{-4}$ and $\lambda_2 = 1 \times 10^{-5}$ [S1, unofficial].
5.  **Optimization:**
    *   Use Adam optimizer [S1, unofficial].
    *   Learning rates: 0.02 for grid parameters, 0.002 for MLP parameters [S1, unofficial].
    *   Iterate for 3000 iterations [S1, unofficial].
6.  **Alpha Mask Updates:**
    *   Update alpha masks at iterations 1500 and 2500 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_tensorf_aert(views, poses, iterations=3000, batch_size=2048):
    # 1. Initialize Density Grid (VM Rank-16)
    # P_i: density feature planes, L_i: density feature lines
    P = initialize_grid(rank=16, shape=(16, 16, 16))
    L = initialize_lines(rank=16)
    
    # 2. Initialize Appearance MLP (2x128 ReLU)
    mlp = initialize_mlp(hidden_units=[128, 128], activation='relu')
    
    # 3. Set Optimizers
    optimizer_grid = Adam(params=P + L, lr=0.02)
    optimizer_mlp = Adam(params=mlp.parameters(), lr=0.002)
    
    # 4. Training Loop
    for step in range(iterations):
        # Sample batch of rays
        rays = sample_rays(views, poses, batch_size)
        
        # Query Density
        # f_theta(x) from VM decomposition of P and L
        f_theta = query_vm_decomposition(rays.x, P, L)
        
        # Corrected Density Parameterization (Delta=0, s=1)
        # Avoids zero gradient flow caused by default Delta=-10
        sigma = relu(f_theta) 
        
        # Query Color
        c = mlp(rays.x, rays.d, f_theta) # Assuming 27-dim features passed or derived
        
        # Render
        I_hat = volume_render(rays, sigma, c)
        
        # Compute Losses
        L_photo = mse(I_hat, rays.I)
        L_sparse = sum(abs(P)) # L1 on density planes
        L_TV = sum(abs(horizontal_diff(P))) + sum(abs(vertical_diff(P)))
        
        L = L_photo + 1e-4 * L_sparse + 1e-5 * L_TV
        
        # Backpropagate
        optimizer_grid.zero_grad()
        optimizer_mlp.zero_grad()
        L.backward()
        
        optimizer_grid.step()
        optimizer_mlp.step()
        
        # Alpha mask updates
        if step in [1500, 2500]:
            update_alpha_masks(rays, sigma)
```

## Subtopics

### Raw TensoRF with Default Parameterization
Uses the original TensoRF density parameterization with $\Delta = -10$ and $s = 25$ [S1, unofficial]. Trained on 72 MedNeRF-rendered views [S1, unofficial]. Achieves a training PSNR of 32.0 dB, confirming the model learns to reproduce projected views [S1, unofficial]. However, the Marching Cubes sweep remains unstable (e.g., 204% error at p=97), revealing that high photometric fidelity does not imply geometrically consistent volumetric structure [S1, unofficial]. This variant serves as a baseline to demonstrate the necessity of the density-shift correction [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Converges in approximately 8 minutes per patient on an NVIDIA T4 GPU [S1, unofficial].
*   **Memory Complexity:** The density grid is $16^3$ voxels with rank-16 VM decomposition; the appearance MLP is small (2x128) [S1, unofficial].
*   **Strengths:** The corrected parameterization ($\Delta=0, s=1$) ensures stable gradient flow and geometric consistency, reducing Marching Cubes error to ~13.8-14.3% compared to >200% with default parameters [S1, unofficial]. Anatomy-aware regularization ($L_{sparse}, L_{TV}$) effectively prevents overfitting in sparse-view scenarios [S1, unofficial].
*   **Limitations:** Highly sensitive to threshold selection during Marching Cubes extraction if not properly regularized; default parameterization leads to complete training failure (zero density) [S1, unofficial].
*   **When to use:** Sparse-view volumetric reconstruction where geometric consistency is critical and standard NeRF/TensoRF fails to produce stable isosurfaces [S1, unofficial].

## Background
Originated from the AReT (Anatomy-Regularized TensoRF) method for sparse-view lung nodule volumetry from DRRs [S1, unofficial]. The work highlights that standard TensoRF density parameterization ($\Delta=-10, s=25$) causes the ReLU to be identically zero at initialization, halting gradient flow [S1, unofficial]. The proposed correction ($\Delta=0, s=1$) and anatomy-aware regularization (L1 sparsity and TV smoothness) are tailored to the thoracic cavity's air-filled nature and nodule morphology [S1, unofficial].

## Sources
[S1] Sparse-View Lung Nodule Volumetry from Digitally Reconstructed Radiographs via AReT: Anatomy-Regularized TensoRF, https://doi.org/10.48550/arxiv.2606.02639, fetched 2026-10-26 (unofficial)
