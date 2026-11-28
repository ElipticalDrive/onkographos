# 2D Covariance Correction via Linear Regression

Also called: DARBF covariance approximation, Scalar correction factor $\psi$, Scalar correction factor $ψ$

## Overview
This method approximates the relationship between projected 2D covariances and actual 2D covariances using a linear regression model, serving as a computationally efficient alternative to neural network-based corrections in differentiable rendering pipelines [S1]. It sets itself apart from sibling methods like [2D Gaussian Covariance Stabilization and Rescaling](3467-2d-gaussian-covariance-stabilization-and-rescaling.md) by deriving explicit linear weights from data rather than applying fixed geometric scaling or stabilization heuristics, thereby correcting projection errors with higher fidelity at the cost of a small, fixed linear transformation overhead [S1].

## How it works
**Inputs and Outputs:**
The input is the 3D covariance matrix $\Sigma'$ of a Gaussian primitive, assumed to be symmetric. The output is the corrected 2D covariance matrix $\Sigma'_{2 \times 2}$, which is a $2 \times 2$ symmetric matrix used for rasterization.

**Data Structures:**
The 3D covariance matrix $\Sigma'$ is represented as a $3 \times 3$ symmetric matrix. Due to symmetry, it is defined by 6 unique scalar values:
$$ \Sigma' = \begin{pmatrix} a & b & c \\ b & d & e \\ c & e & f \end{pmatrix} $$
*Note: The source text [S1] contains a typo in the matrix representation, listing the element at (3,2) as $d$ instead of $e$. The standard symmetric definition and the regression formulas below confirm the element is $e$. The variables $a, b, c, d, e, f$ correspond to $\Sigma'_{00}, \Sigma'_{01}, \Sigma'_{02}, \Sigma'_{11}, \Sigma'_{12}, \Sigma'_{22}$ respectively.*

**Numbered Steps:**
1.  **Extract 3D Covariance Elements:** Retrieve the 6 unique elements $a, b, c, d, e, f$ from the input 3D covariance matrix $\Sigma'$.
2.  **Compute Corrected 2D Elements:** Calculate the three unique elements of the output 2D covariance matrix ($a', b', d'$) using the linear regression formulas derived from training on random samples of 3D covariances [S1].
3.  **Construct 2D Covariance Matrix:** Assemble the corrected elements into the $2 \times 2$ matrix:
    $$ \Sigma'_{2 \times 2} = \begin{pmatrix} a' & b' \\ b' & d' \end{pmatrix} $$

**Formulas:**
The corrected elements are computed as follows [S1]:

$$ a' = 1.36a - 0.2b - 0.1c - 0.0d + 0.0e + 0.0f $$
$$ b' = 0.0a + 1.36b - 0.0c - 0.0d + 0.0e + 0.0f $$
$$ d' = 0.0a + 0.0b - 0.2c + 1.36d + 0.1e + 0.0f $$

These can be simplified by observing that only the direct diagonal elements and specific off-diagonal terms have non-zero weights. The source further notes that this linear relationship can be approximated by a scalar correction factor $\psi$ applied to the direct 2D projection, though the explicit linear form above is the primary implementation detail provided [S1]:
$$ \Sigma'_{2 \times 2} \approx \psi \begin{pmatrix} a & b \\ b & d \end{pmatrix} $$
However, the full linear regression provides distinct weights for cross-terms ($b, c, e$), making the explicit formulas above the accurate implementation.

**Parameters:**
*   **Regression Coefficients:** The weights ($1.36, -0.2, -0.1, 0.1$) are fixed constants derived from training on 1000 random samples of 3D covariances [S1].
*   **Learning Rate (LR):** Used during the training phase of the regression model. Tested values are 0.01, 0.02, 0.1, and 0.2 [S1]. The optimal LR for the Gaussian kernel was found to be 0.02, yielding the highest PSNR [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def correct_covariance_linear_regression(sigma_3d):
    """
    Corrects a 3D covariance matrix to a 2D covariance matrix using linear regression weights.
    
    Args:
        sigma_3d: A 3x3 symmetric numpy array or list of lists representing the 3D covariance.
                  sigma_3d = [[a, b, c],
                                [b, d, e],
                                [c, e, f]]
    
    Returns:
        sigma_2d: A 2x2 numpy array or list of lists representing the corrected 2D covariance.
                  sigma_2d = [[a_prime, b_prime],
                                [b_prime, d_prime]]
    """
    # Extract unique elements from the 3x3 symmetric matrix
    a = sigma_3d[0][0]
    b = sigma_3d[0][1]
    c = sigma_3d[0][2]
    d = sigma_3d[1][1]
    e = sigma_3d[1][2]
    f = sigma_3d[2][2]
    
    # Apply linear regression weights derived from training
    # a' = 1.36a - 0.2b - 0.1c
    a_prime = 1.36 * a - 0.2 * b - 0.1 * c
    
    # b' = 1.36b
    b_prime = 1.36 * b
    
    # d' = -0.2c + 1.36d + 0.1e
    d_prime = -0.2 * c + 1.36 * d + 0.1 * e
    
    # Construct and return the 2x2 corrected covariance matrix
    sigma_2d = [[a_prime, b_prime],
                [b_prime, d_prime]]
    
    return sigma_2d
```

**Edge Cases and Pitfalls:**
*   **Precision:** The coefficients are small decimals; standard floating-point precision (float32) is sufficient, but care should be taken with very small covariance values to avoid underflow if the matrix is near-singular.
*   **Symmetry:** The input matrix must be symmetric. If the 3D covariance is not symmetric due to numerical errors, the extraction of $b, c, e$ should use the average of symmetric pairs or enforce symmetry before correction.
*   **Learning Rate Sensitivity:** During the training of the regression model (not the inference step), excessively high or low learning rates lead to suboptimal performance. At LR=0.1 and 0.2, PSNR drops significantly compared to LR=0.02 [S1].

## Operating characteristics
**Time and Memory Complexity:**
The correction step is $O(1)$ with a fixed number of arithmetic operations (9 multiplications and 6 additions for the full linear form). It is significantly faster than training or evaluating a neural network [S1].

**Measured Performance:**
Evaluated on the Mip-NeRF 360 dataset (scenes: Kitchen, Room, Truck, Bicycle, Flowers, Garden, Stump, Treehill, Counter, Bonsai) [S1]:
*   At LR=0.02 (optimal for Gaussian kernel):
    *   PSNR: 31.75 (Gaussian) vs 31.34 (Half-Cosine/HCS) [S1].
    *   Training Time: 19:48 (Gaussian) vs 17:32 (HCS) [S1].
    *   Memory: 293.61 (Gaussian) vs 282.65 (HCS) [S1].
*   At LR=0.1:
    *   PSNR: 31.1 (Gaussian) vs 29.38 (HCS) [S1].
    *   Training Time: 13:31 (Gaussian) vs 11:20 (HCS) [S1].
    *   Memory: 364 (Gaussian) vs 391.76 (HCS) [S1].

**Scaling:**
Performance scales with hardware capabilities. On an RTX 4090 compared to an RTX 3090, training times are reduced (e.g., Room scene: 19m 48s -> 17m 32s for Gaussian) [S1].

**Strengths and Limitations:**
*   **Strengths:** Provides higher PSNR than the Half-Cosine (HCS) kernel at optimal learning rates (e.g., 31.75 vs 31.34 at LR=0.02) [S1]. The linear model is simple and fast to evaluate [S1].
*   **Limitations:** Cosine (HCS) kernels achieve better training speeds than Gaussian kernels even on upgraded hardware (RTX 4090) [S1]. Cosine kernels do not induce smoothness to the same extent as Gaussian kernels, and at high learning rates, they perform worse [S1].

**When to Use:**
Use when high reconstruction fidelity (PSNR) is prioritized over training speed, and the Gaussian kernel is preferred for its smoothness properties. Use when computational resources allow for the slightly longer training times associated with the Gaussian kernel [S1].

**When Not to Use:**
Do not use if training speed is the primary constraint; the Half-Cosine (HCS) kernel is faster [S1]. Do not use if the learning rate cannot be carefully tuned, as performance degrades significantly at non-optimal rates (e.g., LR=0.2 yields PSNR of 30.5) [S1].

## Background
**Origin:**
The method was introduced in "DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions" [S1]. It was developed to approximate the relationship between projected 2D covariances (obtained by integrating along one axis using Simpson's rule) and actual 2D covariances (derived by removing the last row/column of the 3D covariance matrix) [S1].

**History and Development:**
Initially, a single-layer Multi-Layer Perceptron (MLP) was used to model this relationship. However, extensive experiments showed that the MLP could be closely approximated by a linear regression model, allowing for a simplified scalar correction factor or explicit linear weights [S1]. The method was evaluated against prior methods like Plenoxels, INGP, Mip-NeRF360, and GES, showing competitive PSNR and SSIM scores [S1].

**Usage:**
It is used in 3D Gaussian Splatting variants that employ Decaying Anisotropic Radial Basis Functions (DARBF) to improve rendering quality and reconstruction accuracy [S1].

## Sources
[S1] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
