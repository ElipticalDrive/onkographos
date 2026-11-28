# Covariance Matrix (CM) Splatting

Also called: Plane Principal Component Analysis

## Overview
Covariance Matrix (CM) Splatting is a surface-aligned splatting technique that orients elliptical surface patches by solving the eigenvalue and eigenvector problems associated with a covariance matrix computed from local neighbor points [S1, unofficial]. It aligns splats with the direction of minimal curvature, providing shape-aware, rotationally invariant orientations [S1, unofficial]. Unlike the Householder (HH) method, which computes globally coherent tangent fields without considering surface curvature, CM relies solely on local geometric statistics, resulting in less coherent fields and more discontinuities between splats [S1, unofficial].

## How it works
**Inputs:** A set of 3D points $P = \{p_i\}$ with associated precomputed normal vectors $n_i$, and a kernel radius $r = 5\epsilon$, where $\epsilon$ is the estimated average point distribution [S1, unofficial].
**Outputs:** For each splat center $c$, a normal vector $v_0$, two tangent vectors $v_1, v_2$, and eigenvalues $\lambda_0, \lambda_1, \lambda_2$ defining the splat's orientation and scale [S1, unofficial].

**Data Structures:**
*   **Covariance Matrix $C$**: A $3 \times 3$ symmetric matrix.
*   **Eigenvalues $\lambda$**: A vector of three scalar values, sorted such that $\lambda_0 \le \lambda_1 \le \lambda_2$ [S1, unofficial].
*   **Eigenvectors $v$**: A set of three orthogonal unit vectors corresponding to $\lambda_0, \lambda_1, \lambda_2$ [S1, unofficial].

**Steps:**
1.  **Identify Neighbors:** For a splat center $c$, identify all points $p_i$ within the kernel radius $r = 5\epsilon$ [S1, unofficial].
2.  **Compute Centroid:** Calculate the centroid $c_{local}$ of the neighbor points:
    $$ c_{local} = \frac{1}{N} \sum_{i=1}^{N} p_i $$
    where $N$ is the number of neighbors [S1, unofficial].
3.  **Compute Covariance Matrix:** Calculate the covariance matrix $C$ using the average squared sum of distances between the neighbor points and the centroid:
    $$ C = \frac{1}{N} \sum_{i=1}^{N} (p_i - c_{local})(p_i - c_{local})^T $$
    [S1, unofficial].
4.  **Solve Eigenproblem:** Compute the eigenvalues $\lambda_0, \lambda_1, \lambda_2$ and corresponding eigenvectors $v_0, v_1, v_2$ of $C$ [S1, unofficial].
5.  **Assign Vectors:**
    *   The eigenvector $v_0$ corresponding to the **lowest** eigenvalue $\lambda_0$ is assigned as the estimated normal vector [S1, unofficial].
    *   The eigenvectors $v_1$ and $v_2$ corresponding to the **greatest** eigenvalues $\lambda_1$ and $\lambda_2$ are assigned as the two tangent vectors [S1, unofficial].
6.  **Define Splat Size:** Use the eigenvalues $\lambda_1$ and $\lambda_2$ to define the size (scale) of the splat along the tangent directions [S1, unofficial].

**Parameters:**
*   **Kernel Size:** $5\epsilon$, where $\epsilon$ is the estimated average point distribution [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Noise Sensitivity:** CM reacts poorly to noise. Even in close neighborhoods with similar normal vectors, noise causes increasing variations in tangent vector directions [S1, unofficial].
*   **Discontinuities:** The method creates more discontinuities (holes) between splats compared to Householder splatting [S1, unofficial].
*   **Local Coherence:** Because CM aligns splats only locally, the resulting tangent fields are less coherent than those produced by HH [S1, unofficial].

## Subtopics

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Local angular deviation of the tangent vector fields was measured on several models with varying noise levels ($\epsilon_0, \epsilon_1, \epsilon_2$). Lower deviation indicates better coherence with precomputed normals.

| Model | Noise Level | CM Deviation ($\epsilon_0$) | CM Deviation ($\epsilon_1$) | CM Deviation ($\epsilon_2$) |
| :--- | :--- | :--- | :--- | :--- |
| Armadillo | $\epsilon_0$ | 2.29, 7.62 | - | - |
| Armadillo | $\epsilon_1$ | - | 17.03, 63.35 | - |
| Armadillo | $\epsilon_2$ | - | - | 20.06, 69.69 |
| Buddah | $\epsilon_0$ | 2.43, 9.94 | - | - |
| Buddah | $\epsilon_1$ | - | 15.49, 57.89 | - |
| Buddah | $\epsilon_2$ | - | - | 15.74, 58.06 |
| Bunny | $\epsilon_0$ | 2.31, 8.26 | - | - |
| Bunny | $\epsilon_1$ | - | 13.04, 50.56 | - |
| Bunny | $\epsilon_2$ | - | - | 18.61, 62.77 |
| Dragon | $\epsilon_0$ | 2.30, 7.88 | - | - |
| Dragon | $\epsilon_1$ | - | 16.73, 63.28 | - |
| Dragon | $\epsilon_2$ | - | - | 21.54, 68.95 |
| Bee | $\epsilon_0$ | 2.22, 11.49 | - | - |
| Bee | $\epsilon_1$ | - | 19.73, 68.12 | - |
| Vase | $\epsilon_0$ | 1.77, 4.96 | - | - |
| Vase | $\epsilon_1$ | - | 15.06, 59.01 | - |
| Cathedral | $\epsilon_0$ | 1.06, 18.67 | - | - |
| Cathedral | $\epsilon_1$ | - | 12.74, 49.37 | - |
| Male | $\epsilon_0$ | 3.04, 15.96 | - | - |
| Male | $\epsilon_1$ | - | 14.28, 51.95 | - |

*Note: The table in the source lists two values for each CM entry (e.g., "2.29, 7.62"). The source does not explicitly define what these two numbers represent, but they are consistently paired with CM results [S1, unofficial].*

**Strengths and Limitations:**
*   **Strengths:** Rotationally invariant; produces shape-aware directions; eigenvalues allow for dynamic resizing of splats [S1, unofficial].
*   **Limitations:** Poor performance on noisy data; less coherent tangent fields than Householder; creates more discontinuities; strictly local alignment [S1, unofficial].

**When to use/not use:**
Use CM when the input data is clean and rotational invariance is required [S1, unofficial]. Avoid CM for noisy data (e.g., depth sensor scans) where Householder splatting provides significantly lower angular deviation and better coherence [S1, unofficial].

## Background
CM Splatting was introduced as a standard method for surface-aligned splatting in non-photorealistic rendering [S1, unofficial]. It relies on Plane Principal Component Analysis [Rus09] to estimate splat normal vectors and orientations from 3D point data [S1, unofficial]. It was compared against the Householder (HH) formula in a study by Anjos et al. [dARLP17] to evaluate rendering quality on noisy meshes and point clouds [S1, unofficial].

## Sources
[S1] To Splat Straight with Crooked Points: Rendering Noisy Meshes and Point Clouds using Coherent Tangent Vector Fields, https://doi.org/10.2312/egp.20211028, fetched 2026-10-26 (unofficial)
