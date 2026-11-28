# Weighted Prior Based Reconstruction

Also called: Tomographic Reconstruction to Detect New Structures, Weighted Prior

## Overview
Weighted Prior Based Reconstruction is a tomographic reconstruction method that uses spatially varying weights to moderate the local influence of prior templates, allowing for the detection and accurate reconstruction of new structures in longitudinal scans while suppressing prior dominance in unchanged regions [S1, unofficial]. Unlike existing prior-based methods that impose an inflexible constant weight, this method computes regional weights that temper the role of priors, preventing the suppression of new features in the test object [S1, unofficial].

## How it works
**Inputs:**
*   $y$: Measured tomographic projections of the test volume.
*   $\{Q_i\}_{i=1}^L$: A set of $L$ high-quality prior template volumes.
*   $\Phi$: Matrix modelling the measurement geometry (projection operator).
*   $\Psi$: Basis in which data is assumed to be sparse.
*   $\lambda_1, \lambda_2$: Tunable weights for sparsity and prior terms.
*   $k$: Scaling constant for the weight function.

**Outputs:**
*   $x$: The reconstructed volume.
*   $W$: Spatially varying weight matrix.

**Data Structures:**
*   $x$: 3D volume array.
*   $W$: 3D volume array of same dimensions as $x$, containing scalar weights.
*   $\theta$: Sparse coefficient vector such that $x = \Psi \theta$.
*   $\alpha$: Eigen-coefficient vector for the prior subspace.
*   $\mu$: Mean vector of the template set.
*   $V_i$: $i$-th principal component (eigenvector) of the template set.

**Formulas:**
The reconstruction minimizes the energy function [S1, unofficial]:
$$ E(\theta, \alpha) = \|\Phi x - y\|_2^2 + \lambda_1 \|\theta\|_1 + \lambda_2 \|W (x - (\mu + \sum_i \alpha_i V_i))\|_2^2 $$
where $x = \Psi \theta$. The unknowns $\theta$ and $\alpha$ are solved via alternate minimization [S1, unofficial]. $\theta$ is solved using a basis pursuit Compressed Sensing (CS) solver [S1, unofficial].

**Algorithm Steps:**

1.  **Preliminary Test Reconstruction:** Perform a preliminary reconstruction $X_{fdk}$ of the test volume using the Feldkamp-Davis-Kress (FDK) algorithm on the measured projections $y$ [S1, unofficial].

2.  **Compute Low-Quality Template Volumes:**
    a. For each template $Q_i$ ($i=1 \dots L$), generate simulated measurements $y_{Qi}$ using the exact same projection views and imaging geometry as the test measurements $y$ [S1, unofficial].
    b. Perform preliminary FDK reconstructions of each template $Q_i$ from its simulated measurements $y_{Qi}$. Let this set be denoted by $\{Y_i^{fdk}\}_{i=1}^L$ [S1, unofficial].

3.  **Build Primary Eigenspace:** Build an eigenspace $E_{low}^{fdk}$ from the set $\{Y_i^{fdk}\}_{i=1}^L$. Let $P^{fdk}$ denote the projection of the preliminary test reconstruction $X_{fdk}$ onto $E_{low}^{fdk}$ [S1, unofficial].

4.  **Multi-Method Pilot Reconstructions:**
    a. Perform pilot reconstructions of the test volume using $M$ different reconstruction algorithms. Let this set be denoted as $\{X_j\}_{j=1}^M$, where $X_1 = X_{fdk}$ [S1, unofficial].
    b. For each template $Q_i$ and each reconstruction algorithm $j$ ($j=1 \dots M$), perform a reconstruction of $Q_i$ using the simulated measurements $y_{Qi}$. Let this set be denoted by $\{Y_{ij}\}_{j=1}^M$ for each $i$, where $Y_{i1} = Y_i^{fdk}$ [S1, unofficial].
    c. For each reconstruction algorithm $j$, build an eigenspace $E_{low}^j$ from the set of reconstructed templates $\{Y_{1j}, Y_{2j}, \dots, Y_{Lj}\}$ [S1, unofficial].
    d. For each algorithm $j$, project the pilot test reconstruction $X_j$ onto $E_{low}^j$. Let this projection be denoted by $P^j$ [S1, unofficial].

5.  **Compute Discrepancy:** To remove reconstruction-method-dependent false positives, compute the minimum absolute difference across all methods for each voxel $(x, y, z)$ [S1, unofficial]:
    $$ D(x, y, z) = \min_j |X_j(x, y, z) - P^j(x, y, z)| $$

6.  **Compute Weight Matrix:** Calculate the weight $W(x, y, z)$ for each voxel coordinate using the discrepancy $D(x, y, z)$ [S1, unofficial]:
    $$ W(x, y, z) = (1 + k \cdot D(x, y, z))^{-1} $$
    Regions with high discrepancy (potential new structures) receive lower weights, reducing prior influence. Regions with low discrepancy (unchanged) receive weights closer to 1, allowing prior dominance [S1, unofficial].

7.  **Final Reconstruction:** Solve the optimization problem $E(\theta, \alpha)$ using the computed weight matrix $W$ to obtain the final volume $x$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.linalg import svd

def weighted_prior_reconstruction(y, templates_Q, Phi, Psi, lambda1, lambda2, k, M_methods):
    """
    y: measured projections of test volume
    templates_Q: list of L template volumes
    Phi: projection matrix
    Psi: sparsity basis matrix
    lambda1, lambda2: regularization weights
    k: weight scaling constant
    M_methods: list of reconstruction functions, M_methods[0] is FDK
    """
    L = len(templates_Q)
    
    # Step 1: Preliminary test reconstruction using FDK
    X_fdk = M_methods[0](y)
    
    # Step 2: Compute low-quality template volumes using FDK
    Y_fdk_list = []
    for Q_i in templates_Q:
        # Simulate measurements for template using same geometry
        y_Qi = Phi @ Q_i.flatten()
        # Reconstruct template from simulated measurements
        Y_i_fdk = M_methods[0](y_Qi)
        Y_fdk_list.append(Y_i_fdk)
    
    # Step 3: Build eigenspace from low-quality templates
    # Flatten templates for PCA
    Y_fdk_matrix = np.array([Y.flatten() for Y in Y_fdk_list])
    # Center data
    Y_fdk_mean = np.mean(Y_fdk_matrix, axis=0)
    Y_fdk_centered = Y_fdk_matrix - Y_fdk_mean
    # Compute eigenspace (using SVD for stability)
    U, S, Vt = svd(Y_fdk_centered, full_matrices=False)
    # Use all components or a subset; sources imply full space spanned by templates
    E_low_fdk = Vt.T 
    
    # Project X_fdk onto eigenspace
    X_fdk_flat = X_fdk.flatten()
    X_fdk_centered = X_fdk_flat - Y_fdk_mean
    P_fdk_flat = E_low_fdk @ (E_low_fdk.T @ X_fdk_centered)
    P_fdk = P_fdk_flat.reshape(X_fdk.shape)
    
    # Step 4: Multi-method pilot reconstructions
    X_pilots = []
    Y_pilots = [] # List of lists: Y_pilots[i][j] is template i reconstructed by method j
    
    for j, recon_func in enumerate(M_methods):
        # 4a: Reconstruct test volume
        X_j = recon_func(y)
        X_pilots.append(X_j)
        
        Y_pilots_j = []
        for i, Q_i in enumerate(templates_Q):
            y_Qi = Phi @ Q_i.flatten()
            Y_ij = recon_func(y_Qi)
            Y_pilots_j.append(Y_ij)
        Y_pilots.append(Y_pilots_j)
        
        # 4c: Build eigenspace for method j
        Y_pilots_j_matrix = np.array([Y.flatten() for Y in Y_pilots_j])
        Y_pilots_j_mean = np.mean(Y_pilots_j_matrix, axis=0)
        Y_pilots_j_centered = Y_pilots_j_matrix - Y_pilots_j_mean
        U_j, S_j, Vt_j = svd(Y_pilots_j_centered, full_matrices=False)
        E_low_j = Vt_j.T
        
        # 4d: Project X_j onto eigenspace j
        X_j_flat = X_j.flatten()
        X_j_centered = X_j_flat - Y_pilots_j_mean
        P_j_flat = E_low_j @ (E_low_j.T @ X_j_centered)
        P_j = P_j_flat.reshape(X_j.shape)
        X_pilots[j] = X_j # Store for later use if needed, but we need P_j
        # We need to store P_j for step 5. Let's store it in a list.
        # Note: The pseudocode structure above modifies X_pilots but we need P_j.
        # Let's restructure slightly to keep P_j.
        
    # Re-run Step 4d properly to collect P_j
    P_pilots = []
    for j, recon_func in enumerate(M_methods):
        X_j = X_pilots[j]
        Y_pilots_j = Y_pilots[j]
        Y_pilots_j_matrix = np.array([Y.flatten() for Y in Y_pilots_j])
        Y_pilots_j_mean = np.mean(Y_pilots_j_matrix, axis=0)
        Y_pilots_j_centered = Y_pilots_j_matrix - Y_pilots_j_mean
        U_j, S_j, Vt_j = svd(Y_pilots_j_centered, full_matrices=False)
        E_low_j = Vt_j.T
        
        X_j_flat = X_j.flatten()
        X_j_centered = X_j_flat - Y_pilots_j_mean
        P_j_flat = E_low_j @ (E_low_j.T @ X_j_centered)
        P_j = P_j_flat.reshape(X_j.shape)
        P_pilots.append(P_j)

    # Step 5: Compute discrepancy
    # Initialize discrepancy volume with infinity
    discrepancy = np.full(X_fdk.shape, np.inf)
    
    for j in range(len(M_methods)):
        X_j = X_pilots[j]
        P_j = P_pilots[j]
        diff = np.abs(X_j - P_j)
        discrepancy = np.minimum(discrepancy, diff)
        
    # Step 6: Compute weight matrix
    W = 1.0 / (1.0 + k * discrepancy)
    
    # Step 7: Final Reconstruction
    # Solve E(theta, alpha) = ||Phi Psi theta - y||^2 + lambda1 ||theta||_1 + lambda2 ||W (Psi theta - (mu + sum alpha_i V_i))||^2
    # This requires an optimization solver. The source states theta is solved using basis pursuit CS solver
    # and alpha is solved via alternate minimization.
    # Implementation of the specific solver is not detailed in the source beyond "basis pursuit CS solver".
    # We assume a function solve_optimization exists that handles this specific energy function.
    
    # Compute mu and V_i from original high-quality templates Q
    Q_matrix = np.array([Q.flatten() for Q in templates_Q])
    mu_flat = np.mean(Q_matrix, axis=0)
    Q_centered = Q_matrix - mu_flat
    U_q, S_q, Vt_q = svd(Q_centered, full_matrices=False)
    # V_i are the columns of U_q (or rows of Vt_q transposed)
    # The source uses V_i as principal components.
    # Let's assume V_i corresponds to the eigenvectors of the template covariance.
    
    # Placeholder for the actual optimization step as the specific CS solver implementation details are not provided
    # in the source text beyond the name.
    x_final = solve_weighted_prior_optimization(y, Phi, Psi, W, mu_flat, Vt_q.T, lambda1, lambda2)
    
    return x_final, W

def solve_weighted_prior_optimization(y, Phi, Psi, W, mu, V_components, lambda1, lambda2):
    """
    Placeholder for the alternate minimization solver.
    The source specifies:
    1. theta is solved using basis pursuit CS solver.
    2. alpha is solved (implied least squares or similar given the quadratic term).
    3. Alternate minimization.
    """
    # This part is not fully specified in the source.
    # We return a dummy value to satisfy the "no placeholders" rule for the method's core definition.
    # However, the source does not give the algorithm for the solver, only that it uses "basis pursuit CS solver".
    # We must indicate this is not given.
    raise NotImplementedError("The specific algorithm for solving the alternate minimization problem is not given in the sources.")
```

**Edge Cases and Pitfalls:**
*   **Alignment:** If alignment is not present between scans, all template volumes must be pre-aligned before computing the eigenspace, and the test volume must be registered to the templates after its preliminary pilot reconstruction [S1, unofficial].
*   **False Positives:** The use of multiple reconstruction methods and taking the minimum difference helps remove reconstruction-method-dependent false positives [S1, unofficial].

## Operating characteristics
*   **Complexity:** The method requires multiple preliminary reconstructions (FDK and $M$ other methods) for both the test volume and $L$ templates. This increases computational cost compared to single-pass reconstruction.
*   **Performance:** Validated on synthetic and real volume data, specifically biological specimens in longitudinal studies (Potato dataset with 5% projection views, Okra dataset with 10% projection views) [S1, unofficial].
*   **Strengths:** Significantly improves overall quality of reconstructed data while minimizing the impact of priors on regions containing new information [S1, unofficial].
*   **Limitations:** Requires a set of high-quality prior templates. Performance depends on the accuracy of the preliminary reconstructions and the choice of $M$ methods.

## Background
*   **Origin:** The method was introduced in "Learning from past scans: Tomographic reconstruction to detect new structures" [S1, unofficial].
*   **History:** Prior-based methods using eigenspaces were shown to better accommodate variation in the match of a test volume to a set of templates [S1, unofficial]. However, global priors impose inflexible constant weights, leading to suppression of new features in longitudinal studies [S1, unofficial]. This method addresses that limitation by computing regional weights [S1, unofficial].
*   **Usage:** Longitudinal studies where the same object is scanned multiple times to monitor for small changes over time [S1, unofficial].

## Sources
[S1] Learning from past scans: Tomographic reconstruction to detect new structures, https://arxiv.org/abs/1812.10998v1, fetched 2026-10-26 (unofficial)
