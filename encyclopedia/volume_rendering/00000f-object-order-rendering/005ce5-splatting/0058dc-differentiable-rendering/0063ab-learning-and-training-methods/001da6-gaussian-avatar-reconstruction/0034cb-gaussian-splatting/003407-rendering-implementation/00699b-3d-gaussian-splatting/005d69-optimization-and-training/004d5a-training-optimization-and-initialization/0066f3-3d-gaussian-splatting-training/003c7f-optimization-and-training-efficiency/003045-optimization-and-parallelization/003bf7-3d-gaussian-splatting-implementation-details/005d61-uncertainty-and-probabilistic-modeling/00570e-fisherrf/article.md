# FisherRF

Also called: Active View Selection and Mapping with Radiance Fields Using Fisher Information

## Overview
FisherRF is a method for active view selection and uncertainty quantification in Radiance Fields that directly quantifies observed information on model parameters using Fisher Information, selecting candidate views by maximizing Expected Information Gain (EIG) [S1]. It sets itself apart from sibling methods like [SOGS](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md) or [Gaussian Frosting](15863-gaussian-frosting.md) by focusing on the *selection* of new training views rather than just optimizing existing parameters, leveraging the sparsity of the Hessian to achieve real-time evaluation speeds (70 fps) [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A trained or partially trained Radiance Field model (specifically 3D Gaussian Splatting or Plenoxels), a pool of candidate camera viewpoints, and the current set of training views.
- **Output:** A ranked list of candidate views based on their Expected Information Gain (EIG), or a selected subset of views for active mapping.

**Data Structures:**
- **Hessian Matrix ($H$):** Represents the second-order derivatives of the loss function with respect to the model parameters. In 3D Gaussian Splatting, this is computed per Gaussian.
- **Fisher Information Matrix (FIM):** Approximated by the Hessian of the negative log-likelihood.

**Method Steps:**
1. **Compute Hessian:** For the current model state, compute the Hessian matrix of the parameters with respect to the negative log-likelihood for each 3D Gaussian [S1]. The Hessian of the objective function in volumetric rendering is independent of ground truth data or actual image measurements, depending only on the camera parameters of the views used for training [S1].
2. **Approximate Information Gain:** To make computation tractable, approximate the decrease in entropy (Information Gain) by its upper bound. This bound is defined as the trace of the product of the Hessian of the candidate view ($H_{cand}$) and the Hessian with respect to all previous views ($H_{prev}$) [S1]:
   $$ \text{EIG} \approx \text{Tr}(H_{cand} \cdot H_{prev}) $$
3. **Leverage Sparsity:** The matrices are highly sparse due to the disentangling of scene parameters with respect to disparate rays. This sparsity allows for a cost-effective computation of the matrix trace, with complexity comparable to back-propagation [S1].
4. **Rank and Select:** Evaluate all candidate views using the approximation. Select the view(s) with the highest EIG for the next iteration of training or mapping [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_fisher_information_gain(model, candidate_views, training_views):
    """
    Computes the Expected Information Gain for a set of candidate views
    based on the Fisher Information (Hessian) of the current model.
    
    Args:
        model: A 3D Gaussian Splatting or Plenoxels model instance.
        candidate_views: List of camera poses to evaluate.
        training_views: List of camera poses already used for training.
        
    Returns:
        List of tuples (view, score) sorted by score descending.
    """
    # Step 1: Compute Hessian for current training views
    # The Hessian is independent of ground truth, relying only on camera params.
    # In 3DGS, this is computed per Gaussian.
    H_prev = model.compute_hessian(training_views)
    
    scores = []
    
    for view in candidate_views:
        # Step 2: Compute Hessian for the candidate view
        H_cand = model.compute_hessian([view])
        
        # Step 3: Approximate EIG as Trace(H_cand * H_prev)
        # Due to sparsity, this trace computation is efficient (~back-prop cost)
        # [G] Assuming H_prev and H_cand are sparse matrices or diagonal approximations
        # where the trace of the product can be computed element-wise or via sparse dot products.
        eig = trace_product(H_cand, H_prev)
        
        scores.append((view, eig))
        
    # Step 4: Sort by EIG
    scores.sort(key=lambda x: x[1], reverse=True)
    return scores

def trace_product(H1, H2):
    """
    Computes the trace of the product of two sparse matrices.
    Tr(A * B) = sum_i sum_j A_ij * B_ji
    For sparse matrices, this is optimized by iterating over non-zero elements.
    """
    # [G] Implementation detail: Iterate over non-zero entries of H1.
    # If H1 and H2 are diagonal (common approximation in 3DGS for efficiency),
    # Tr(H1 * H2) = sum(H1_diag * H2_diag).
    # The source notes mention "diagonal Hessian computation" in comparison to Goli et al.
    return sum(h1 * h2 for h1, h2 in zip(H1.diagonal(), H2.diagonal()))
```

**Parameters:**
- **Performance Metric:** The method evaluates views at 70 fps when using 3D Gaussian Splatting [S1].
- **Hessian Computation Time:** 11.3 ms ± 33.3 µs for the diagonal Hessian computation [S1].

**Edge Cases and Pitfalls:**
- **Sparsity Assumption:** The efficiency relies on the high sparsity of the Hessian due to the disentangling of scene parameters. If the scene representation is dense or parameters are highly coupled, the trace computation may become expensive [S1].
- **Approximation Error:** The use of the trace of the product as an upper bound for entropy decrease is an approximation. It may not perfectly capture the true Information Gain in all scenarios, though it is stated to be effective [S1].

## Operating characteristics
- **Time Complexity:** The computation of the trace is as cost-effective as back-propagation due to matrix sparsity [S1].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Evaluates views at 70 fps with 3D Gaussian Splatting [S1]. The Hessian computation takes 11.3 ms ± 33.3 µs, compared to 1.1 s ± 2.5 ms for alternative methods like Goli et al. [S1].
- **Strengths:** Direct quantification of uncertainty on model parameters; efficient CUDA implementation for diagonal Hessian computation; state-of-the-art results in view selection and active mapping [S1].
- **Limitations:** Relies on the sparsity of the Hessian; uses an approximation for Information Gain [S1].
- **When to use:** When active view selection or active mapping is required for Radiance Fields, particularly 3D Gaussian Splatting or Plenoxels [S1].
- **When not to use:** When the scene representation is dense and sparsity cannot be exploited, or when exact Information Gain is required over computational speed [S1].

## Background
FisherRF was introduced by W. Jiang et al. in the paper "FisherRF: Active View Selection and Mapping with Radiance Fields Using Fisher Information" [S1]. It addresses the problem of active "training" view selection for volumetric rendering, a problem that was extensively studied before radiance fields but lacked literature specific to them [S1]. The method builds on the framework of 3D Gaussian Splatting and Plenoxels, leveraging their differentiable nature and parameter structure [S1]. It improves upon previous methods like Goli et al. by computing the Hessian directly rather than relying on hypothetical perturbation fields and multiple backward passes [S1].

## Sources
[S1] FisherRF: Active View Selection and Mapping with Radiance Fields Using Fisher Information, https://doi.org/10.1007/978-3-031-72624-8_24, fetched 2026-10-31
