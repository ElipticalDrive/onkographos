# CBARF
Also called: CBARF (Cascaded Bundle-Adjusting Neural Radiance Fields), Cascaded BA

## Overview
CBARF is a 3D reconstruction model that optimizes imperfect camera poses using a cascaded bundle-adjustment (BA) structure to progressively refine poses and rectify erroneous ones [S1]. It builds on [BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md) by introducing a coarse-to-fine optimization strategy with distinct stages, allowing it to handle larger initial pose errors (e.g., 30.4° rotation deviation) where standard BARF may overfit or diverge [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of images with imperfect camera poses $\hat{P}$, a NeRF backbone (typically BARF), and ground-truth images for photometric loss.
*   **Outputs:** Refined camera poses $Q$, optimized NeRF parameters, and synthesized images.

**Data Structures:**
*   **Camera Pose:** Represented as $P \in \mathbb{R}^{3 \times 4}$, defined as $P = [R_p \ t_p]$ where $R_p \in SO(3)$ and $t_p \in \mathbb{R}^3$ [S1]. The last row of the $4 \times 4$ transformation matrix is dropped [S1].
*   **Lie Algebra Noise:** Perturbations are represented as $n \in \mathbb{R}^6$ from the Lie algebra $\mathfrak{se}(3)$ [S1].

**Optimization Steps:**

1.  **Initialization with Noise:**
    Simulate imperfect poses by adding noise to reference poses. Generate noise $n$ from a 6-dimensional normal distribution:
    $$ n = 0.35 \mathcal{N}(0, I) \in \mathbb{R}^6 $$
    [S1]. This corresponds to an average rotation deviation of $30.4^\circ$ and translation deviation of $0.56$ [S1]. Transform $n$ into a transform matrix $N = [R_n \ t_n]$ and compose with the reference pose $P$ to get imperfect poses $\hat{P}$:
    $$ \hat{P} = [R_p \ R_n \ t_p + t_n] $$
    [S1].

2.  **Cascaded BA Structure:**
    Use a coarse-to-fine structure with compact BA modules, using BARF as the backbone [S1]. The process consists of three stages: Coarse, Recursive, and Fine [S1].

3.  **Pose Refinement:**
    In each stage, optimize camera refine parameters $p \in \mathfrak{se}(3)$. Convert $p$ to a transform matrix $P_r = [R_r \ t_r]$ and compose with the current imperfect pose $\hat{P}$ to obtain refined poses $Q$:
    $$ Q = [R_p \ R_n \ R_r \ t_p + t_n + t_r] $$
    [S1].

4.  **Learning Rate Schedule:**
    Use exponential interpolation with a modulation factor $\alpha$ to bias learning towards the initial high value [S1]. The modulation factors for the Coarse, Recursive, and Fine stages are $10.0$, $3.0$, and $1.0$ respectively [S1]. A factor of $1.0$ corresponds to the original exponential learning rate [S1].

5.  **Incomplete Poses Handling:**
    For test sets with missing poses, randomly drop 10% of poses, assign an initial pose $P_{ini}$ to them, and jointly optimize poses for both training and test sets [S1].

6.  **Evaluation Alignment:**
    Use Procrustes analysis to align optimized poses with ground-truth before evaluation to remove global offset [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cbarf_optimize(images, imperfect_poses, nerf_model, stages):
    """
    stages: list of dicts with keys 'iterations', 'modulation_factor', 'resolution'
    """
    current_poses = imperfect_poses
    nerf_params = nerf_model.parameters()
    
    for stage in stages:
        iterations = stage['iterations']
        alpha = stage['modulation_factor']
        res = stage['resolution']
        
        # Initialize optimizer for current stage
        # Note: Specific optimizer details (e.g., Adam) are not specified in [S1]
        optimizer = create_optimizer([nerf_params, current_poses])
        
        for step in range(iterations):
            # Calculate learning rate with modulation
            lr = calculate_lr(step, iterations, alpha)
            optimizer.set_lr(lr)
            
            # Sample rays at reduced resolution
            rays = generate_rays(current_poses, width=res, height=res)
            
            # Render
            rendered_colors = nerf_model(rays)
            
            # Compute loss (MSE/LPIPS)
            loss = compute_loss(rendered_colors, images)
            
            # Backpropagate
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
            
            # Update poses via refinement parameters p in se(3)
            # The specific update rule for p -> Pr -> Q is handled by the optimizer
            # on the pose parameters, which are initialized as noise in se(3)
            
        # After stage, current_poses are updated to Q
        current_poses = refine_poses(current_poses)
        
    return current_poses, nerf_model

def calculate_lr(step, total_steps, modulation_factor):
    """
    Exponential interpolation with modulation factor.
    If modulation_factor == 1.0, uses standard exponential decay.
    """
    # Standard exponential decay formula from BARF/NeRF literature [S1]
    # Base learning rate is not specified in [S1], assumed to be standard NeRF value
    base_lr = 0.0001 
    decay = 0.95 # Example decay rate, not explicitly given in [S1]
    
    # Modulation factor biases the learning rate towards the initial value
    # Formula: lr = base_lr * (decay ^ (step / total_steps * modulation_factor))
    # Note: Exact formula for modulation is not explicitly written in [S1], 
    # but described as "biasing learning towards the initial high value".
    # Implementation detail [G]: Assuming standard exponential scaling with exponent scaled by alpha.
    lr = base_lr * (decay ** (step / total_steps * modulation_factor))
    return lr

def refine_poses(poses):
    """
    Compose current poses with the learned refinement transform Pr.
    poses: [R_p R_n t_p + t_n]
    """
    # Get the refinement parameters p from the optimizer state
    p = get_refinement_params() # p in se(3)
    
    # Convert p to matrix Pr
    Pr = se3_to_matrix(p) # R_r, t_r
    
    # Compose: Q = P * Pr
    # Note: [S1] defines Q = [Rp Rn Rr tp + tn + tr]
    # This implies right-multiplication by the refinement matrix if P is [R t]
    # However, standard composition is T_new = T_old * T_refine
    # We assume the optimizer updates the parameters such that the composition is handled correctly.
    return compose_poses(poses, Pr)
```

**Parameters:**
*   **Noise Coefficient:** $0.35$ for CBARF (vs $0.15$ in BARF) [S1].
*   **Iterations:** $20,000$ for Coarse and Recursive stages; $200,000$ for Fine stage [S1].
*   **Resolution:** $400 \times 400$ during pose optimization (half of $800 \times 800$ training resolution) [S1].
*   **Modulation Factors:** $10.0$ (Coarse), $3.0$ (Recursive), $1.0$ (Fine) [S1].

**Edge Cases and Pitfalls:**
*   **Overfitting:** Standard BARF may overfit before completing optimization with large initial errors; CBARF mitigates this via cascaded stages [S1].
*   **Coordinate Misalignment:** Optimized poses may have a global offset relative to ground truth; Procrustes analysis is required for evaluation [S1].
*   **Incomplete Poses:** Randomly dropping 10% of poses requires joint optimization of training and test set poses [S1].

## Operating characteristics
*   **Time Complexity:** Higher than standard NeRF due to the cascaded structure and multiple optimization stages (Coarse, Recursive, Fine) [S1].
*   **Memory Complexity:** Similar to BARF, as it uses the same backbone and reduces image resolution during optimization ($400 \times 400$) [S1].
*   **Performance:** Successfully optimizes poses with $30.4^\circ$ rotation deviation, whereas BARF fails [S1]. Rendering quality is comparable to ground-truth poses [S1].
*   **Strengths:** Handles large initial pose errors; robust to incomplete poses; rectifies erroneous poses via cascaded refinement [S1].
*   **Limitations:** Requires careful tuning of modulation factors and iteration counts per stage; evaluation requires Procrustes alignment [S1].

## Background
CBARF was proposed in 2024 by authors publishing in the IEEE Transactions on Multimedia [S1]. It extends the BARF method [S1] by addressing the limitation of standard bundle adjustment in handling large initial pose errors through a cascaded coarse-to-fine strategy.

## Sources
[S1] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
