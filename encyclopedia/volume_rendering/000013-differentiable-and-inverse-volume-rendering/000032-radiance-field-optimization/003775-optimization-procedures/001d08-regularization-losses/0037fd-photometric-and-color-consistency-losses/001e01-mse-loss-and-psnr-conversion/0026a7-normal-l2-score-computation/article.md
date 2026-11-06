# Normal-L2 Score Computation
Also called: Masked Normal L2, Normal Error Metric
## Overview
The Normal-L2 score evaluates surface geometry by computing the mean squared L2 distance between predicted surface normals and reference normals from a ground-truth mesh [S1]. It is computed only on the intersection of predicted and ground-truth masks, distinguishing it from silhouette metrics like iIoU which evaluate mask alignment separately [S1].
## How it works
**Inputs:**
- Predicted surface normals $N_{pred}$: A tensor of shape $(H, W, 3)$ containing normalized normal vectors for each pixel [G].
- Reference surface normals $N_{ref}$: A tensor of shape $(H, W, 3)$ containing interpolated normal vectors from a ground-truth mesh [S1, G].
- Predicted mask $M_{pred}$: A binary tensor of shape $(H, W)$ indicating pixels belonging to the predicted surface [G].
- Ground-truth mask $M_{gt}$: A binary tensor of shape $(H, W)$ indicating pixels belonging to the ground-truth surface [G].

**Outputs:**
- Normal-L2 score: A scalar value representing the mean squared error of the normal vectors over the valid region [S1, G].

**Data Structures:**
- Masks are boolean or float tensors where 1 indicates surface presence and 0 indicates background [G].
- Normals are 3D unit vectors [G].

**Steps:**
1. **Compute Intersection Mask:** Calculate the intersection of the predicted and ground-truth masks to ensure normals are only compared where both surfaces exist [S1, G]. In implementation, this is performed via logical AND or element-wise multiplication followed by a threshold check [G].
2. **Extract Valid Normals:** Select normal vectors from $N_{pred}$ and $N_{ref}$ where $M_{valid}$ is true [G]. This is typically done via boolean indexing or coordinate filtering [G].
3. **Compute Squared Differences:** Calculate the squared Euclidean distance between corresponding valid normal vectors [G].
   $$ D_{uv} = \| N_{pred}^{valid}(u,v) - N_{ref}^{valid}(u,v) \|^2 $$
   $$ D_{uv} = (n_{pred,x} - n_{ref,x})^2 + (n_{pred,y} - n_{ref,y})^2 + (n_{pred,z} - n_{ref,z})^2 $$
   [G] Normals must be normalized before this step to ensure unit length [G].
4. **Average:** Compute the mean of $D_{uv}$ over all valid pixels [S1, G].
   $$ \text{Normal-L2} = \frac{1}{|M_{valid}|} \sum_{(u,v) \in M_{valid}} D_{uv} $$
   [G] If $|M_{valid}| = 0$, the metric returns 0 or raises an error depending on convention [G].

**Parameters:**
- Camera positions: 32 evenly distributed fixed positions using a spherical Fibonacci sequence with radius 4 [S1].
- Render resolution: 512x512 [S1].
- Samples per pixel: 4 [S1].
- Renderer: Mitsuba 2 is used for mesh rendering [S1].

**Edge Cases and Pitfalls:**
- **Mask Misalignment:** If masks do not overlap, the denominator becomes zero [S1]. The metric is only evaluated on the intersection [S1].
- **Normal Orientation:** Normals must be consistently oriented (e.g., outward-facing) to avoid large errors due to sign flips [G].
- **Numerical Stability:** Normals should be normalized before comparison to ensure unit length [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal_l2_score(n_pred, n_ref, mask_pred, mask_gt):
    """
    Compute Normal-L2 score.
    
    Args:
        n_pred: List of lists of shape (H, W, 3), predicted normals.
        n_ref: List of lists of shape (H, W, 3), reference normals.
        mask_pred: List of lists of shape (H, W), predicted mask (1.0 for surface, 0.0 otherwise).
        mask_gt: List of lists of shape (H, W), ground-truth mask (1.0 for surface, 0.0 otherwise).
        
    Returns:
        float: Normal-L2 score.
    """
    h = len(mask_pred)
    w = len(mask_pred[0])
    
    valid_count = 0
    sum_sq_diff = 0.0
    
    for y in range(h):
        for x in range(w):
            if mask_pred[y][x] > 0.5 and mask_gt[y][x] > 0.5:
                dx = n_pred[y][x][0] - n_ref[y][x][0]
                dy = n_pred[y][x][1] - n_ref[y][x][1]
                dz = n_pred[y][x][2] - n_ref[y][x][2]
                sum_sq_diff += dx*dx + dy*dy + dz*dz
                valid_count += 1
                
    if valid_count == 0:
        return 0.0
    return sum_sq_diff / valid_count
```
## Subtopics

### KL Normal Loss
Computes the Kullback-Leibler divergence between a standard normal distribution and a diagonal Gaussian parameterized by the mean and variance of an output embedding tensor. It is used to regularize the latent space of implicit surface models rather than evaluating surface geometry.
- **Definition:** Measures the divergence of the distribution of latent embeddings from a standard normal prior.
- **Steps:** 1. Compute mean $\mu_d$ and standard deviation $\sigma_d$ along dimension 0 for each dimension $d$ of the output embedding tensor. 2. Compute variance $\sigma_d^2 = \text{std}_d^2$. 3. Compute KL per dimension: $kl_d = 0.5 (\mu_d^2 + \sigma_d^2 - \ln(\sigma_d^2 + \epsilon) - 1)$. 4. Sum over all dimensions: $KL = \sum_d kl_d$.
- **Formula:** $$ KL = \sum_{d} 0.5 \left( \mu_d^2 + \sigma_d^2 - \ln(\sigma_d^2 + \epsilon) - 1 \right) $$ where $\mu_d$ is the mean along dimension 0 for dimension $d$, $\sigma_d^2$ is the variance, and $\epsilon = 1e-8$ for numerical stability.
- **When to choose:** Use when regularizing latent embeddings in generative models rather than evaluating surface geometry.
- **Link:** [KL Normal Loss](2742-kl-normal-loss.md)

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Measured Performance:** Evaluated at 512x512 resolution with 4 samples per pixel [S1].
- **Strengths:** Sensitive to surface orientation errors, which are perceptually important for shading [S1].
- **Limitations:** Requires ground-truth mesh normals for comparison, which may not always be available. Sensitive to mask alignment errors [S1].
- **When to use:** When evaluating geometric accuracy of implicit surface reconstructions, especially when shading quality is important [S1].
## Background
The metric was used alongside intersection over union (iIoU) to evaluate shape silhouettes and surface normals, as these drive shading and are perceptually important [S1]. It is implemented using Mitsuba 2 for mesh rendering [S1].
## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] TurtleJayWang/VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
