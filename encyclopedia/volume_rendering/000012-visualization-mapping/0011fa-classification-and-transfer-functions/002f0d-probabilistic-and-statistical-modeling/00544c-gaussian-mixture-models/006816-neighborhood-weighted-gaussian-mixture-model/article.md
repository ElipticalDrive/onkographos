# Neighborhood Weighted Gaussian Mixture Model

Also called: Gaussian Mixture Model with Neighborhood Weighting, NW-GMM, Neighborhood-weighted GMM, Weighted Gaussian Mixture Model

## Overview
The Neighborhood Weighted Gaussian Mixture Model (NW-GMM) is a statistical classification method for vectorial volumes that integrates spatial neighborhood information into the Gaussian mixture model to solve misclassification on boundaries and inhomogeneous regions with noise [S1][S2, unofficial]. It extends the standard Gaussian Mixture Model (GMM) by weighting class probabilities with the probabilities of neighboring voxels, thereby enforcing spatial continuity [S1][S2, unofficial]. Unlike standard GMM, which relies solely on voxel intensity distributions and is susceptible to noise and partial volume effects, NW-GMM produces more homogeneous regions and is less sensitive to noise [S1][S2, unofficial]. It builds on [Gaussian Mixture Models](15189-gaussian-mixture-models.md) by modifying the probability estimation step within the Expectation-Maximization (EM) algorithm.

## How it works
**Inputs and Outputs:**
*   **Input:** A vectorial volume composed of $I$ voxels, where each voxel $x_i$ is an $n$-dimensional vector representing $n$ registered acquisitions (e.g., multi-phase CT scans) [S1][S2, unofficial].
*   **Input:** $K$, the number of tissues or materials (classes) [S1][S2, unofficial].
*   **Input:** $N_i$, the chosen neighborhood set of voxels surrounding voxel $i$ [S2, unofficial].
*   **Output:** Class probabilities $p(k|x_i, \Theta)$ for each voxel $i$ and class $k=1 \dots K$ [S1][S2, unofficial].
*   **Output:** Model parameters $\Theta$ (means, covariances) and mixing coefficients [S1][S2, unofficial].

**Data Structures:**
*   **Vectorial Volume:** A 3D grid of size $D_x \times D_y \times D_z$. Each element is a vector of size $n$.
*   **Parameters $\Theta_k$:** For each class $k$, a mean vector $\mu_k$ of size $n$ and a covariance matrix $\Sigma_k$ of size $n \times n$ [S2, unofficial].
*   **Mixing Coefficients:** A vector $\alpha$ (or $\beta$) of size $K$, where $\sum_{k=1}^K \alpha_k = 1$ [S1][S2, unofficial].
*   **Probabilities:** A 3D grid of size $D_x \times D_y \times D_z \times K$, storing $p(k|x_i, \Theta)$ for each voxel and class.
*   **Neighborhood Weights:** A temporary 3D grid of size $D_x \times D_y \times D_z \times K$, storing $W_{ik}$ for each voxel and class.

**Algorithm Steps:**
The method uses the Expectation-Maximization (EM) algorithm to estimate parameters and mixing coefficients, with a modified E-step that incorporates neighborhood weights [S1][S2, unofficial].

1.  **Initialization:** Initialize parameters $\Theta^{(0)}$ (means $\mu_k$, covariances $\Sigma_k$) and mixing coefficients $\alpha^{(0)}_k$ for $k=1 \dots K$. Initialize class probabilities $p(k|x_i, \Theta^{(0)})$ for all voxels $i$ and classes $k$. [G] (Standard initialization: random or k-means).
2.  **Iteration:** Repeat until convergence (change in parameters or log-likelihood is below threshold $\epsilon$):
    a.  **E-step (Expectation with Neighborhood Weighting):**
        i.   For each voxel $i$ and class $k$, calculate the neighborhood weight $W_{ik}$:
        $$ W_{ik} = \sum_{n \in N_i} p(k|x_n, \Theta^{(t)}) $$
        where $N_i$ is the set of neighbor indices for voxel $i$, and $p(k|x_n, \Theta^{(t)})$ is the class probability from the previous iteration (or current iteration if using Gauss-Seidel style updates, but sources imply using previous iteration's probabilities for weight calculation) [S2, unofficial].
        ii.  For each voxel $i$ and class $k$, calculate the weighted posterior probability:
        $$ p(k|x_i, \Theta^{(t)}) = \frac{\alpha_k W_{ik} f_k(x_i | \Theta^{(t)})}{\sum_{j=1}^{K} \alpha_j W_{ij} f_j(x_i | \Theta^{(t)})} $$
        where $f_k(x_i | \Theta^{(t)})$ is the multivariate Gaussian probability density function for class $k$ at voxel $x_i$ [S1][S2, unofficial].
        iii. Note: The denominator in the source formula [S2, unofficial] uses $W_{ij}$ in the sum, implying the weight is specific to the class $j$ being summed over in the denominator. The formula is:
        $$ p(k|x_i, \Theta^{(t)}) = \frac{\alpha_k W_{ik} f_k(x_i | \Theta^{(t)})}{\sum_{j=1}^{K} \alpha_j W_{ij} f_j(x_i | \Theta^{(t)})} $$
        This ensures the probability is normalized across classes for voxel $i$ [S1][S2, unofficial].
    b.  **M-step (Maximization):**
        i.   Update the mixing coefficients $\alpha_k^{(t+1)}$ using the weighted responsibilities (probabilities) $p(k|x_i, \Theta^{(t)})$. Standard GMM M-step formulas apply, but weighted by the neighborhood-influenced probabilities [S1][S2, unofficial].
        $$ \alpha_k^{(t+1)} = \frac{1}{I} \sum_{i=1}^{I} p(k|x_i, \Theta^{(t)}) $$
        ii.  Update the parameters $\Theta_k^{(t+1)}$ (mean $\mu_k$ and covariance $\Sigma_k$) for each class $k$ using the weighted responsibilities [S1][S2, unofficial].
        $$ \mu_k^{(t+1)} = \frac{\sum_{i=1}^{I} p(k|x_i, \Theta^{(t)}) x_i}{\sum_{i=1}^{I} p(k|x_i, \Theta^{(t)})} $$
        $$ \Sigma_k^{(t+1)} = \frac{\sum_{i=1}^{I} p(k|x_i, \Theta^{(t)}) (x_i - \mu_k^{(t+1)})(x_i - \mu_k^{(t+1)})^T}{\sum_{i=1}^{I} p(k|x_i, \Theta^{(t)})} $$
3.  **Output:** Return the final class probabilities $p(k|x_i, \Theta)$ and parameters $\Theta$ [S1][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def neighborhood_weighted_gmm(vectorial_volume, K, neighborhood_indices, max_iter=100, tol=1e-4):
    """
    vectorial_volume: ndarray of shape (D, H, W, n), n-dimensional vectors per voxel
    K: int, number of classes
    neighborhood_indices: dict or list of lists, mapping voxel index to list of neighbor indices
    max_iter: int, maximum number of EM iterations
    tol: float, convergence tolerance
    """
    I = vectorial_volume.shape[0] * vectorial_volume.shape[1] * vectorial_volume.shape[2]
    n = vectorial_volume.shape[3]
    
    # Flatten volume for easier indexing
    voxels = vectorial_volume.reshape(-1, n)
    
    # Initialize parameters [G]
    # Means: random selection of K voxels
    indices = np.random.choice(I, K, replace=False)
    mu = voxels[indices].copy()
    # Covariances: Identity matrices
    Sigma = np.array([np.eye(n) for _ in range(K)])
    # Mixing coefficients: uniform
    alpha = np.ones(K) / K
    
    # Initialize probabilities
    # p[i, k] = probability of class k for voxel i
    p = np.zeros((I, K))
    
    for t in range(max_iter):
        # --- E-step ---
        
        # 1. Calculate likelihoods f_k(x_i) for all voxels and classes
        # Multivariate Gaussian PDF
        # f_k(x) = (2*pi)^(-n/2) * |Sigma_k|^(-1/2) * exp(-0.5 * (x-mu_k)^T Sigma_k^-1 (x-mu_k))
        likelihoods = np.zeros((I, K))
        for k in range(K):
            diff = voxels - mu[k] # (I, n)
            # Mahalanobis distance squared
            # (x-mu)^T Sigma^-1 (x-mu)
            # Efficiently compute for all voxels
            inv_Sigma = np.linalg.inv(Sigma[k])
            temp = diff @ inv_Sigma # (I, n)
            mahal = np.sum(temp * diff, axis=1) # (I,)
            
            # Determinant and normalization constant
            sign, log_det = np.linalg.slogdet(Sigma[k])
            norm_const = -0.5 * n * np.log(2 * np.pi) - 0.5 * log_det
            
            likelihoods[:, k] = np.exp(norm_const - 0.5 * mahal)
            
        # 2. Calculate neighborhood weights W_ik
        # W_ik = sum_{n in N_i} p(k|x_n, Theta_t)
        # Note: p is from previous iteration t
        W = np.zeros((I, K))
        for i in range(I):
            neighbors = neighborhood_indices[i]
            for k in range(K):
                W[i, k] = np.sum(p[neighbors, k])
                
        # 3. Calculate weighted posterior probabilities p(k|x_i, Theta_t)
        # p(k|x_i) = (alpha_k * W_ik * f_k(x_i)) / sum_j (alpha_j * W_ij * f_j(x_i))
        numerator = alpha * W * likelihoods # (I, K)
        denominator = np.sum(numerator, axis=1, keepdims=True) # (I, 1)
        
        # Avoid division by zero
        denominator = np.maximum(denominator, 1e-10)
        
        p_new = numerator / denominator # (I, K)
        
        # Check convergence
        if np.max(np.abs(p_new - p)) < tol:
            p = p_new
            break
        p = p_new
        
        # --- M-step ---
        
        # Update mixing coefficients
        # alpha_k = (1/I) * sum_i p(k|x_i)
        alpha = np.mean(p, axis=0)
        
        # Update means and covariances
        for k in range(K):
            # Effective number of points for class k
            N_k = np.sum(p[:, k])
            if N_k < 1e-10:
                continue # Skip empty class
            
            # Update mean
            mu[k] = np.sum(p[:, k, np.newaxis] * voxels, axis=0) / N_k
            
            # Update covariance
            diff = voxels - mu[k] # (I, n)
            weighted_diff = diff * p[:, k, np.newaxis] # (I, n)
            # Sigma_k = (1/N_k) * sum_i p(k|x_i) * (x_i - mu_k)(x_i - mu_k)^T
            Sigma[k] = (weighted_diff.T @ diff) / N_k
            
    return p, mu, Sigma, alpha

```

**Parameters:**
*   $K$: Number of tissues/materials. Typical values depend on the medical imaging application (e.g., 3-10) [S1][S2, unofficial].
*   $n$: Dimensionality of the voxel vector, equal to the number of registered acquisitions [S1][S2, unofficial].
*   $N_i$: Neighborhood set. Typically a 3x3x3 or 5x5x5 voxel window [S2, unofficial]. Not given in the sources.
*   $\epsilon$: Convergence threshold. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Misclassification:** Standard GMM fails here due to intensity mixing; NW-GMM addresses this by using neighbor probabilities [S1][S2, unofficial].
*   **Noise Sensitivity:** Standard GMM is sensitive to noise; NW-GMM is less sensitive due to spatial smoothing via neighborhood weights [S1][S2, unofficial].
*   **Partial Volume Effects:** NW-GMM handles these better than standard GMM [S2, unofficial].
*   **Empty Classes:** If a class has no voxels with significant probability, the M-step update for that class may fail (division by zero). A guard is needed [G].
*   **Covariance Singularity:** If a class has very few voxels or they are collinear, the covariance matrix may become singular. Regularization (adding a small value to the diagonal) is recommended [G].
*   **Neighborhood Boundary:** Voxels on the volume boundary have fewer neighbors. The neighborhood set $N_i$ must be defined to handle boundaries (e.g., by excluding out-of-bounds neighbors) [G].

## Subtopics

### Gaussian Sum Transfer Function
Uses a sum of Gaussians to define transfer functions directly, rather than probabilistic classification. Chosen when direct opacity/color mapping is preferred over soft segmentation. [Gaussian Sum Transfer Function](1812-gaussian-sum-transfer-function.md)

### Local Data Models for Probabilistic Transfer Function Design
Fits local statistical models to data subsets. Chosen when global GMM assumptions are violated and local adaptability is needed. [Local Data Models for Probabilistic Transfer Function Design](5862-local-data-models-for-probabilistic-transfer-function-design.md)

### Splat Regression
Uses splatting for regression-based classification. Chosen for differentiable rendering pipelines. [Splat Regression](20292-splat-regression.md)

### Gabor Fields
Uses Gabor filters for texture-based classification. Chosen for texture analysis in volumes. [Gabor Fields](22711-gabor-fields.md)

## Operating characteristics
*   **Time Complexity:** $O(I \cdot K \cdot n^3 \cdot T)$, where $I$ is the number of voxels, $K$ is the number of classes, $n$ is the dimensionality of the voxel vector, and $T$ is the number of EM iterations. The $n^3$ comes from matrix inversion in the Gaussian PDF calculation. The neighborhood weight calculation adds $O(I \cdot |N| \cdot K)$ per iteration, where $|N|$ is the neighborhood size [S1][S2, unofficial].
*   **Memory Complexity:** $O(I \cdot K + I \cdot |N| \cdot K)$ for storing probabilities and neighborhood weights. The vectorial volume itself is $O(I \cdot n)$ [S1][S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of voxels and classes. The neighborhood weight calculation scales with neighborhood size.
*   **Strengths:** Less sensitive to noise and partial volume effects than standard GMM. Produces more homogeneous regions. Handles vectorial data. [S1][S2, unofficial]
*   **Limitations:** More computationally expensive than standard GMM due to neighborhood weight calculation. Requires defining a neighborhood. [S1][S2, unofficial]
*   **When to Use:** Medical imaging vectorial volumes for soft segmentation of tissues after 3D/3D registration, especially when noise and boundary misclassification are concerns. [S1][S2, unofficial]
*   **When Not to Use:** When computational resources are limited and noise is low. When scalar data is used and spatial continuity is not critical.

## Background
The Neighborhood Weighted Gaussian Mixture Model was proposed to address the limitations of standard GMM in classifying vectorial volumes, specifically misclassification on boundaries and sensitivity to noise [S1][S2, unofficial]. It was presented in a previous paper [14] by the authors of [S1] and [S2, unofficial]. The method integrates neighborhood information into the EM algorithm by weighting class probabilities with the probabilities of neighboring voxels, based on the hypothesis that tissues are continuous [S2, unofficial]. It is applied to medical imaging vectorial volumes for soft segmentation of tissues after 3D/3D registration [S1][S2, unofficial].

## Sources
[S1] A multivolume visualization method for spatially aligned volumes after 3D/3D image registration, https://doi.org/10.1016/j.irbm.2011.01.040, fetched 2026-10-26
[S2] Research of New Medical Volume Visualization Methods: Application in the Kidney Preoperative Planning System, https://theses.hal.science/tel-00355629v1, fetched 2026-10-26 (unofficial)
