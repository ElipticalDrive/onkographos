# Stochastic Tomography

Also called: Random walk tomography, Stochastic solver for tomography, random walk algorithm for tomographic reconstruction

## Overview
Stochastic Tomography is a matrix-free, grid-free iterative solver for tomographic reconstruction that represents the volume estimate as a set of discrete samples rather than a fixed voxel grid [S1]. It minimizes a convex objective function consisting of data fidelity (sum of squared residuals) and a regularization term by using a random walk to explore the solution space and deposit emissivity [S1]. Unlike grid-based methods like SART, it avoids spatial discretization artifacts and allows for easy replacement of regularizers without re-implementing the solver [S1]. It builds on the [General Numerical Optimization Framework for Tomography](3251-general-numerical-optimization-framework-for-tomography.md) by treating the proximal operators via a stochastic sampling process rather than deterministic updates.

## How it works
**Inputs:**
*   $M$ camera views with observations $\mathbf{o} = \{o_1, \dots, o_M\}$.
*   Projection operators $\text{proj}_j(\mathbf{x})$ mapping 3D points to 2D pixel coordinates in image $j$.
*   Visual hull mask (optional, for rejection of samples outside the volume of interest).
*   Parameters: Sample budget $N_{total}$, number of parallel walks $W$, Gaussian mutation width $\sigma$, initial emissivity magnitude $e_d$.

**Outputs:**
*   A set of samples $\{\mathbf{x}_k\}$ with associated emissivities $\{e_k\}$ representing the volume estimate $\tilde{v}(\mathbf{x})$.
*   Residual images $\mathbf{r}_j$ for each camera $j$, initialized to the observed images $\mathbf{o}_j$.

**Data Structures:**
*   **Samples:** A list of tuples $(\mathbf{x}_k, e_k)$, where $\mathbf{x}_k \in \mathbb{R}^3$ and $e_k \in \mathbb{R}$.
*   **Residual Images:** $M$ 2D arrays $\mathbf{r}_j$ of size corresponding to the camera resolution, initialized to $\mathbf{o}_j$.
*   **Splatting Kernel:** A 2D kernel function $K(u, v)$ used to distribute emissivity from a 3D sample to the 2D residual image.

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize residual images $\mathbf{r}_j \leftarrow \mathbf{o}_j$ for all $j=1 \dots M$.
    *   Initialize $W$ random walks. For each walk $w$, pick a starting point $\mathbf{x}_0^{(w)}$ uniformly distributed within the reconstruction volume bounding box.
    *   Set current objective change $\Delta F^{(w)} \leftarrow 0$ (or a large initial value to accept the first sample).

2.  **Random Walk Loop:**
    *   While total accepted samples $< N_{total}$:
        *   For each active walk $w$:
            1.  **Mutation:** Generate a candidate sample $\mathbf{x}_{k}^{(w)}$ by adding a Gaussian offset to the previous sample $\mathbf{x}_{k-1}^{(w)}$:
                $$ \mathbf{x}_{k}^{(w)} = \mathbf{x}_{k-1}^{(w)} + \boldsymbol{\epsilon}, \quad \boldsymbol{\epsilon} \sim \mathcal{N}(0, \sigma^2 \mathbf{I}) $$
                where $\sigma$ is typically 5-10% of the diagonal of the reconstruction volume bounding box [S1].
            2.  **Visual Hull Check:** If $\mathbf{x}_{k}^{(w)}$ projects outside the visual hull for *any* camera (i.e., the pixel is zero in at least one view), reject the sample and set $\mathbf{x}_{k}^{(w)} \leftarrow \mathbf{x}_{k-1}^{(w)}$ [S1].
            3.  **Objective Change Calculation:** Calculate the change in the objective function $\Delta F(\mathbf{x}_{k}^{(w)})$ if the sample were accepted.
                *   Compute tentative residuals for positive and negative emissivity $\pm e_d$:
                    $$ \tilde{\mathbf{r}}_j^{\pm} = \text{splat}(\mathbf{r}_j, \text{proj}_j(\mathbf{x}_{k}^{(w)}), \pm e_d) $$
                    where $\text{splat}$ adds the emissivity to the residual image using the kernel [S1].
                *   Calculate the change in squared residuals for each camera $j$:
                    $$ \Delta_j^{\pm} F(\mathbf{x}_{k}^{(w)}) = \|\mathbf{r}_j(\text{proj}_j(\mathbf{x}_{k}^{(w)}))\|^2 - \|\tilde{\mathbf{r}}_j^{\pm}(\text{proj}_j(\mathbf{x}_{k}^{(w)}))\|^2 $$
                    Note: $\mathbf{r}_j(\text{proj}_j(\mathbf{x}))$ denotes the value at the projected pixel location. The norm here refers to the local contribution to the sum of squares, effectively the difference in the squared residual value at the splatted location [S1].
                *   Sum over all cameras:
                    $$ \Delta^{\pm} F(\mathbf{x}_{k}^{(w)}) = \sum_{j=1}^M \Delta_j^{\pm} F(\mathbf{x}_{k}^{(w)}) $$
                *   Select the best sign:
                    $$ \Delta F(\mathbf{x}_{k}^{(w)}) = \max(\Delta^+ F(\mathbf{x}_{k}^{(w)}), \Delta^- F(\mathbf{x}_{k}^{(w)})) $$
                *   **Regularization:** If a regularizer $\Gamma$ is used, calculate its energy change $\Delta \Gamma(\mathbf{x}_{k}^{(w)})$ based on the stencil centered at $\mathbf{x}_{k}^{(w)}$ (see [Stochastic Tomography Regularization](#stochastic-tomography-regularization)). Add this to the objective change:
                    $$ \Delta F_{total}(\mathbf{x}_{k}^{(w)}) = \Delta F(\mathbf{x}_{k}^{(w)}) + \Delta \Gamma(\mathbf{x}_{k}^{(w)}) $$
            4.  **Acceptance:**
                *   Calculate acceptance probability $a$:
                    $$ a = \min\left(1, \frac{\Delta F_{total}(\mathbf{x}_{k}^{(w)})}{\Delta F^{(w)}}\right) $$
                    where $\Delta F^{(w)}$ is the objective change from the *previous* accepted step in this walk [S1].
                *   If $\Delta F^{(w)} < 0$ (initial step) or $\text{random()} \le a$:
                    *   If $\Delta F_{total}(\mathbf{x}_{k}^{(w)}) > 0$:
                        *   Accept the sample. Record $(\mathbf{x}_{k}^{(w)}, e_{sign})$ where $e_{sign}$ is the sign ($+$ or $-$) that yielded $\Delta F_{total}$.
                        *   Update the residual images $\mathbf{r}_j$ by splatting $\pm e_d$ at $\text{proj}_j(\mathbf{x}_{k}^{(w)})$ [S1].
                        *   Update $\Delta F^{(w)} \leftarrow \Delta F_{total}(\mathbf{x}_{k}^{(w)})$.
                    *   Else:
                        *   Reject. Set $\mathbf{x}_{k}^{(w)} \leftarrow \mathbf{x}_{k-1}^{(w)}$.
                *   Else:
                    *   Reject. Set $\mathbf{x}_{k}^{(w)} \leftarrow \mathbf{x}_{k-1}^{(w)}$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def stochastic_tomography(observations, projections, visual_hull_mask, 
                          num_walks, sample_budget, sigma, emissivity, 
                          regularizer=None):
    """
    observations: List of M 2D arrays (images)
    projections: Function proj(image_idx, point_3d) -> (u, v)
    visual_hull_mask: Function is_in_hull(point_3d) -> bool
    num_walks: W
    sample_budget: N_total
    sigma: Gaussian mutation width
    emissivity: e_d
    regularizer: Function calc_reg_change(prev_samples, new_sample) -> delta_E
    """
    M = len(observations)
    residuals = [img.copy() for img in observations]
    
    # Initialize walks
    # Each walk state: (current_point, prev_delta_F)
    walks = []
    for _ in range(num_walks):
        x0 = np.random.uniform(low=volume_min, high=volume_max, size=3)
        walks.append({'x': x0, 'delta_F': 0.0, 'history': [x0]})
        
    accepted_samples = []
    total_accepted = 0
    
    while total_accepted < sample_budget:
        for w_idx in range(num_walks):
            if total_accepted >= sample_budget:
                break
                
            walk = walks[w_idx]
            x_prev = walk['x']
            delta_F_prev = walk['delta_F']
            
            # 1. Mutation
            epsilon = np.random.normal(0, sigma, 3)
            x_cand = x_prev + epsilon
            
            # 2. Visual Hull Check
            if not visual_hull_mask(x_cand):
                walk['x'] = x_prev
                continue
                
            # 3. Objective Change Calculation
            # Calculate residual change for + and - emissivity
            delta_F_pos = 0.0
            delta_F_neg = 0.0
            
            for j in range(M):
                u, v = projections(j, x_cand)
                # Get current residual value at projection
                r_curr = residuals[j][v, u]
                
                # Simulate splatting +emissivity
                # Note: In practice, splatting affects neighbors. 
                # The source defines delta F as the difference in squared residuals.
                # We approximate the local change.
                # r_new = r_curr + kernel * emissivity
                # delta = r_curr^2 - r_new^2
                
                # For simplicity, assuming point-splat or local kernel effect
                # The source formula: r_j(proj)^2 - r_tilde_j(proj)^2
                # This implies we only care about the pixel value change? 
                # No, "splat" implies distribution. The source says:
                # "splat(r_j, proj(x_k), e_d) returns the resulting image"
                # Then delta F is sum over j of (r_j(proj)^2 - r_tilde_j(proj)^2)
                # This suggests the change is evaluated at the projected pixel location
                # or the integral over the kernel support. 
                # Given the formula (7), it uses r_j(proj_j(x_k))^2. 
                # This likely means the value at the center of the kernel or the 
                # integrated effect. We will assume the change in the residual 
                # image at the projected location is what matters for the delta F calculation 
                # as per the source's simplified formula representation.
                
                # Let's assume a simple point-splat for the delta calculation 
                # to match the formula structure r^2 - (r+e)^2 = -2re - e^2
                # But the source says "splat... returns resulting image".
                # We will implement the delta F as the reduction in the sum of squares 
                # of the residual image.
                
                # To be precise to the source:
                # delta_j = r_j(proj)^2 - r_tilde_j(proj)^2
                # This is ambiguous. Does r_tilde_j(proj) mean the value at the pixel?
                # Yes, likely.
                
                # Calculate new residual value at (u,v) if we splat +e
                # Assuming a kernel K, the new value is sum(K * r_old).
                # If we assume the kernel is normalized and we just add e to the center:
                r_new_pos = r_curr + emissivity # Simplified
                delta_F_pos += r_curr**2 - r_new_pos**2
                
                r_new_neg = r_curr - emissivity
                delta_F_neg += r_curr**2 - r_new_neg**2
                
            delta_F = max(delta_F_pos, delta_F_neg)
            
            # Add Regularization
            delta_reg = 0.0
            if regularizer:
                delta_reg = regularizer(walk['history'], x_cand)
                
            delta_F_total = delta_F + delta_reg
            
            # 4. Acceptance
            if delta_F_prev < 0 or np.random.rand() <= min(1.0, delta_F_total / delta_F_prev):
                if delta_F_total > 0:
                    # Accept
                    sign = '+' if delta_F_pos > delta_F_neg else '-'
                    e_val = emissivity if sign == '+' else -emissivity
                    
                    # Update residuals
                    for j in range(M):
                        u, v = projections(j, x_cand)
                        # Splat e_val into residuals[j]
                        residuals[j] = splat(residuals[j], (u,v), e_val)
                        
                    accepted_samples.append((x_cand, e_val))
                    walk['x'] = x_cand
                    walk['delta_F'] = delta_F_total
                    walk['history'].append(x_cand)
                    total_accepted += 1
                else:
                    walk['x'] = x_prev
            else:
                walk['x'] = x_prev
                
    return accepted_samples, residuals
```

## Subtopics

### Stochastic Tomography Regularization
This variant incorporates prior information into the objective function to handle sparse or noisy measurements [S1]. Regularizers must be convex and have relatively compact support for efficient per-sample evaluation [S1]. They are evaluated on stencils centered at the sample with spacing equal to the grid spacing [S1].
*   **L2 Regularizer:** Uses a 7-point Laplacian stencil in 3D; the result is squared [S1].
*   **Total Variation (TV):** Uses gradient magnitude computed via first-order finite differences [S1].
*   **SAD (Sum of Absolute Differences):** Sums the absolute difference between the center sample and each of the 26 neighbors in a 3x3x3 window [S1].
*   **Image-Space Regularization:** Applies 2D variants of the above regularizers directly on the generated output images. The total regularizer value is the sum of individual image regularizers. This biases the solution to satisfy measurements while producing smooth output, unlike post-processing smoothing which yields results inconsistent with measurements [S1].

## Operating characteristics
*   **Time Complexity:** Computation time scales linearly with the sample budget. For example, 200 million mutations take ~14 minutes per frame, and 400 million mutations take ~31 minutes per frame on a single Intel i7 core [S1].
*   **Memory Complexity:** Matrix-free. Memory usage depends on the number of accepted samples and the size of the residual images, not on the volume resolution grid [S1].
*   **Strengths:** Grid-free representation avoids fixed resolution tradeoffs; easy to swap regularizers without re-implementing the solver; handles sparse views well [S1].
*   **Limitations:** Sensitive to camera misalignment (limits resolution); reflections in the capture environment (e.g., glass beaker walls) can introduce artifacts if consistent across cameras [S1].
*   **When to use:** 3D imaging of mixing fluids, transparent volumes, or scenes where grid-based artifacts are undesirable and sparse views are available [S1].

## Background
Stochastic Tomography was introduced in a paper titled "Stochastic tomography and its applications in 3D imaging of mixing fluids" [S1]. It was developed to address the limitations of grid-based methods like SART, which require fixed output grids and complex re-implementation for different regularizers [S1]. The method was applied to 3D imaging of mixing fluids, including fluorescein-sodium dye in water, alcohol-fluorescein-sodium mixtures, and dyepowder [S1].

## Sources
[S1] Stochastic tomography and its applications in 3D imaging of mixing fluids, https://doi.org/10.1145/2185520.2185548, fetched 2026-11-03
