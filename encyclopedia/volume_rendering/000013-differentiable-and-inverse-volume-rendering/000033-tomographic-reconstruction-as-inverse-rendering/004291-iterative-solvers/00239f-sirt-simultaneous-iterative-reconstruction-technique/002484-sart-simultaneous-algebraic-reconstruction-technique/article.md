# SART (Simultaneous Algebraic Reconstruction Technique)

Also called: Modified SART for Data Term Proximal Operator, PSART SART, SART Data Term Solver, SART for prox_{λ_f}(q), Simultaneous Algebraic Reconstruction Technique

## Overview
SART is an iterative algebraic reconstruction technique based on the Kaczmarz projection method, used to solve the data fidelity term in tomographic reconstruction problems [S1, unofficial]. It updates the reconstructed volume for each scan view, offering modest memory requirements and superior performance over direct methods like FDK in limited projection scenarios [S1, unofficial]. It is preferred over Conjugate Gradient (CG) for sparse-view settings because CG suffers from numerical noise in matrix-free implementations and convergence issues with large sparse systems [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** System matrix $A \in \mathbb{R}^{m \times n}$ (where $m$ is the number of rays and $n$ is the number of voxels), measured projections $p \in \mathbb{R}^m$, previous iterate $q \in \mathbb{R}^n$, regularization parameter $\lambda \in \mathbb{R}$, relaxation parameter $\alpha \in \mathbb{R}$, and total outer iterations $T$.
*   **Outputs:** Reconstructed volume $f \in \mathbb{R}^n$.

**Data Structures:**
*   $A$: Sparse matrix or implicit operator defining ray-voxel intersections $a_{ij}$.
*   $y$: Auxiliary vector of size $m$, initialized to zero.
*   $f$: Volume vector of size $n$.
*   $\hat{b}$: Estimated projection vector of size $m$.
*   $c$: Correction factor vector of size $m$.

**Algorithm Steps:**
The method solves the proximal operator $\text{prox}_{\lambda_f}(q) = \text{argmin}_f \|Af - p\|_2^2 + \frac{1}{2\lambda}\|f - q\|_2^2$ by transforming it into an under-determined linear system using auxiliary variables $y = \sqrt{2\lambda}(p - Af)$ and $z = f - q$ [S1, unofficial].

1.  **Initialization:**
    *   Set $y^{(0)} = 0$.
    *   Set $f^{(0)} = q$.
    *   Scale inputs: $\tilde{p} = \sqrt{2\lambda}p$, $\tilde{A} = \sqrt{2\lambda}A$. [S1, unofficial]

2.  **Outer Loop:** Iterate $t$ from $1$ to $T$.

3.  **Inner Loop:** Iterate over projection sets $S \in \{S_1, \dots, S_N\}$. For each set of rays $S$:
    *   **Forward Projection:** Compute estimated projections for all rays $i$:
        $$ \hat{b}_i^{(t+1)} = \sum_{k} a_{ik} f_k^{(t)} + y_i^{(t)} $$
        [S1, unofficial]
    *   **Correction:** Compute normalized deviations for all rays $i \in S$:
        $$ c_i^{(t+1)} = \frac{b_i - \hat{b}_i^{(t+1)}}{\sum_{k} a_{ik} + 1} $$
        [S1, unofficial]
        *Note: The denominator includes $+1$ due to the augmented system structure involving the auxiliary variable $y$.*
    *   **Update Auxiliary Variable:** Update $y$ for indices $j \in S$:
        $$ y_j^{(t+1)} = y_j^{(t)} + \alpha c_j^{(t)} $$
        [S1, unofficial]
    *   **Backprojection:** Update volume $f$ for all voxels $j=1 \dots n$:
        $$ f_j^{(t+1)} = f_j^{(t)} + \alpha \frac{\sum_{i \in S} c_i^{(t+1)} a_{ij}}{\sum_{i \in S} a_{ij}} $$
        [S1, unofficial]

**Parameters:**
*   $\alpha$: Relaxation parameter. Typical values range from $0.1$ to $0.8$ depending on the dataset (e.g., $0.3$ for Zone plate, $0.1$ for Artificial rose) [S1, unofficial].

**Pitfalls:**
*   **Memory Usage:** High memory consumption occurs if patch-based Jacobians $J_K(f)$ are stored for regularization. For a volume of size $N$, storage can be 82 times the volume size (81 for $J_K(f)$ + 1 for volume). For a $690 \times 668 \times 776$ voxel volume, this requires 109 GB [S1, unofficial].
*   **CG Incompatibility:** Conjugate Gradient is unsuitable for this proximal operator in tomography because: 1) It requires preconditioning for large systems, but storing $A$ is infeasible, limiting preconditioners to ineffective Jacobi types. 2) Matrix-free implementations of $A$ and $A^T$ introduce numerical discrepancies, making $A^T A$ not an exact transpose, which CG is sensitive to [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sart_solve(A_getter, b, q, lam, alpha, T, projection_sets):
    """
    Solves the data term proximal operator using Modified SART.
    
    Args:
        A_getter: Function returning a_ij for ray i and voxel j.
        b: Measured projections p (size m).
        q: Previous iterate (size n).
        lam: Regularization parameter lambda.
        alpha: Relaxation parameter.
        T: Total outer iterations.
        projection_sets: List of sets of ray indices S_1 ... S_N.
        
    Returns:
        f: Reconstructed volume (size n).
    """
    import math
    
    # Determine dimensions
    # Note: In practice, m and n are known from the problem setup.
    # Here we assume they are passed or inferred. 
    # For the sake of self-contained pseudocode, we assume n is len(q)
    # and m is len(b).
    n = len(q)
    m = len(b)
    
    # Initialization
    y = [0.0] * m
    f = list(q) # Copy q
    
    # Scale inputs as per source derivation
    sqrt_2lam = math.sqrt(2 * lam)
    # Note: The source scales p and A, but the algorithm steps 
    # use b_i directly in the correction formula with the augmented denominator.
    # We follow Algorithm 2 in [S1, unofficial] which uses b_i directly 
    # in the correction step c_i = (b_i - b_hat) / (sum(a) + 1).
    # The scaling is implicit in the derivation of the auxiliary variable update.
    
    for t in range(1, T + 1):
        for S in projection_sets:
            # 1. Forward Projection
            # Compute estimated projection sum for all rays
            # b_hat_i = sum_k a_ik * f_k + y_i
            # We compute this for all rays i, but only use correction for i in S
            # However, the source says "Compute estimated projections b_hat_i^(t) for all rays"
            # and then "Compute correction factor c_i^(t+1)" for the current set.
            # To be efficient, we only compute b_hat for rays in S? 
            # The source formula for c_i uses b_hat_i. 
            # Let's compute b_hat for all rays to be safe, or just those in S.
            # The source says "for all rays" in step 1, but step 2 computes c for the set.
            # We will compute b_hat for all rays to match the text description strictly.
            
            b_hat = [0.0] * m
            for i in range(m):
                ray_sum = 0.0
                # Get voxels contributing to ray i
                # A_getter(i, k) returns a_ik
                # Optimization: In real code, iterate over non-zero a_ik
                for k in range(n):
                    a_ik = A_getter(i, k)
                    if a_ik != 0:
                        ray_sum += a_ik * f[k]
                b_hat[i] = ray_sum + y[i]
            
            # 2. Correction & 3. Backprojection & Update y
            # We need to compute corrections for rays in S
            # And update f for all voxels based on rays in S
            
            # Precompute denominator for voxels to avoid repeated summation
            # Denom_j = sum_{i in S} a_ij
            voxel_denom = [0.0] * n
            
            # First pass: Compute corrections and accumulate voxel denominators
            corrections = {} # Map ray index to correction value
            
            for i in S:
                # Compute correction c_i
                # c_i = (b_i - b_hat_i) / (sum_k a_ik + 1)
                # We need sum_k a_ik for ray i
                ray_weight_sum = 0.0
                for k in range(n):
                    a_ik = A_getter(i, k)
                    if a_ik != 0:
                        ray_weight_sum += a_ik
                
                numerator = b[i] - b_hat[i]
                denominator = ray_weight_sum + 1.0
                c_i = numerator / denominator
                corrections[i] = c_i
                
                # Update y for j in S
                # y_j^(t+1) = y_j^(t) + alpha * c_j^(t)
                # Note: The source says "Update y for indices j in current projection set S"
                # and uses c_j^(t). In the loop, c_i is computed for i in S.
                # So we update y[i] using the computed c_i.
                y[i] = y[i] + alpha * c_i
                
                # Accumulate voxel denominators for backprojection
                for k in range(n):
                    a_ik = A_getter(i, k)
                    if a_ik != 0:
                        voxel_denom[k] += a_ik

            # Second pass: Backproject to update f
            for j in range(n):
                if voxel_denom[j] == 0:
                    continue
                
                # Sum of corrections weighted by a_ij for rays in S
                weighted_correction_sum = 0.0
                for i in S:
                    a_ij = A_getter(i, j)
                    if a_ij != 0:
                        weighted_correction_sum += corrections[i] * a_ij
                
                # Update f_j
                f[j] = f[j] + alpha * (weighted_correction_sum / voxel_denom[j])
                
    return f
```

## Operating characteristics
*   **Time Complexity:** Iterative; convergence speed depends on the condition number of the system and the relaxation parameter $\alpha$.
*   **Memory Complexity:** Modest for the core SART algorithm, but can become high if auxiliary structures like patch-based Jacobians are stored (up to 82x volume size) [S1, unofficial].
*   **Strengths:** Handles limited projections better than FDK; allows incorporation of prior information (e.g., TV priors); preferred over CG for sparse-view settings due to CG's sensitivity to numerical noise in matrix-free implementations [S1, unofficial].
*   **Limitations:** Slower convergence than direct methods for full-view data; sensitive to the choice of $\alpha$; memory intensive if complex regularizers requiring Jacobian storage are used [S1, unofficial].
*   **When to use:** Sparse-view tomography, dynamic tomography, or when incorporating non-standard priors in a proximal framework.
*   **When not to use:** Full-view, noise-free data where direct methods (FDK) are sufficient and faster.

## Background
SART was proposed by Andersen et al. [7] as an efficient iterative reconstruction method based on the Kaczmarz projection method [S1, unofficial]. It has been widely used in computed tomography (CT) for its ability to handle limited projection data and incorporate prior information. Recent developments have adapted SART as a solver for the data term proximal operator in proximal optimization frameworks for CT reconstruction, particularly in dynamic and sparse-view scenarios [S1, unofficial].

## Sources
[S1] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
