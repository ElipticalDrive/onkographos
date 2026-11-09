# Gaussian Normal Field Definition

Also called: Definition 3, Gaussian Normal Field, Gaussian vector field, Normalized Gaussian vector field, Proposition 3

## Overview
The Gaussian Normal Field Definition derives an explicit normal field on a stochastic surface from the gradient of the log-vacancy, leveraging the reciprocal attenuation of Oriented Gaussians [S1]. It serves to supervise oriented Gaussians during training to enforce proper surface wrapping and to extract a mesh from the trained representation [S1]. It is distinguished from other analytical vector fields by its specific derivation from the gradient of the log-vacancy in Gaussian Splatting representations, rather than from physical flow or procedural noise functions.

## How it works
**Inputs:**
*   $N$: Number of oriented Gaussians.
*   For each Gaussian $i \in \{1, \dots, N\}$:
    *   $\mu_i \in \mathbb{R}^3$: Mean position.
    *   $n_i \in S^2$: Normal vector (orientation).
    *   $G_i(x)$: The Gaussian function value at position $x$.

**Outputs:**
*   $\mathcal{N}(x) \in \mathbb{R}^3$: The Gaussian normal field at position $x$.

**Data Structures:**
*   The method assumes a list or array of $N$ Gaussian objects, each storing $\mu_i$, $n_i$, and parameters required to evaluate $G_i(x)$ (typically covariance matrix or scale/rotation, though the exact parameterization of $G_i$ is not detailed in the source beyond its existence) [S1].

**Steps:**
1.  **Compute the Gaussian Vector Field $V(x)$**:
    Calculate the gradient of the log-vacancy, defined as the sum of the gradients of the log-vacancy contributions from each Gaussian, masked by the half-space defined by the Gaussian's normal.
    $$
    V(x) = \sum_{i=1}^N \mathbb{1}_{n_i^T(x-\mu_i) \geq 0} \nabla \log(1 - G_i(x))
    $$
    where $\mathbb{1}_{condition}$ is the indicator function (1 if true, 0 otherwise) [S1].

2.  **Normalize to obtain $\mathcal{N}(x)$**:
    Normalize the vector field $V(x)$. If the magnitude is zero, return the zero vector.
    $$
    \mathcal{N}(x) = \begin{cases} \frac{V(x)}{\|V(x)\|} & \text{if } \|V(x)\| > 0 \\ 0 & \text{otherwise} \end{cases}
    $$
    [S1].

**Formulas and Definitions:**
*   $v(x)$: Vacancy.
*   $\nabla \log v(x)$: Gradient of the log-vacancy.
*   $G_i(x)$: The $i$-th Gaussian function.
*   $\nabla \log(1 - G_i(x))$: The gradient of the log-vacancy contribution of the $i$-th Gaussian.
*   $n_i^T(x-\mu_i) \geq 0$: The condition that point $x$ lies in the half-space defined by normal $n_i$ at mean $\mu_i$. This acts as a mask to ensure only the "front" or oriented side of the Gaussian contributes to the gradient sum [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_normal_field(x, gaussians):
    """
    Computes the Gaussian Normal Field at position x.
    
    Args:
        x: numpy array of shape (3,), position in 3D space.
        gaussians: List of dictionaries or objects, each containing:
            - 'mu': numpy array of shape (3,), mean position.
            - 'n': numpy array of shape (3,), normal vector.
            - 'G': callable function G_i(x) returning the Gaussian value at x.
            - 'grad_log_vac': callable function returning grad(log(1 - G_i(x))) at x.
            
    Returns:
        numpy array of shape (3,), the normal vector N(x).
    """
    V = np.zeros(3)
    
    for g in gaussians:
        mu = g['mu']
        n = g['n']
        
        # Check half-space condition: n^T (x - mu) >= 0
        diff = x - mu
        dot_product = np.dot(n, diff)
        
        if dot_product >= 0:
            # Compute gradient of log(1 - G_i(x))
            # Note: The source defines V as sum of these gradients.
            # We assume the helper provides the vector gradient.
            grad_term = g['grad_log_vac'](x)
            V = V + grad_term
            
    norm_V = np.linalg.norm(V)
    
    if norm_V > 0:
        return V / norm_V
    else:
        return np.zeros(3)
```

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Zero Magnitude**: If $\|V(x)\| = 0$, the normal is undefined. The method returns the zero vector [S1]. This can occur in empty space far from any Gaussians or in symmetric configurations where gradients cancel out.
*   **Sign Ambiguity**: The derivation relies on the assumption that locally, each term in the sum has the same sign as the total gradient or is zero. This "disambiguation" of absolute values is valid under the "wrapping assumption" [S1]. If this assumption fails (e.g., complex occlusion or non-wrapped geometry), the formula may not accurately represent the true normal field.
*   **Gaussian Evaluation**: The exact form of $G_i(x)$ and its gradient $\nabla \log(1 - G_i(x))$ is not specified in the source text, relying on standard Gaussian Splatting definitions. Implementers must ensure the gradient computation is numerically stable, especially when $G_i(x)$ is close to 1 (where $\log(1-G_i)$ approaches $-\infty$).

## Subtopics

## Operating characteristics
*   **Time Complexity**: $O(N)$ per evaluation, where $N$ is the number of Gaussians. Each Gaussian requires a dot product, a conditional check, and a gradient evaluation.
*   **Memory Complexity**: $O(N)$ to store the Gaussian parameters ($\mu_i, n_i$). No grid storage is required.
*   **Strengths**: Explicit normal field derivation from Gaussian Splatting representation; enables mesh extraction and training supervision without simulation [S1].
*   **Limitations**: Relies on the "wrapping assumption" for correctness; undefined (zero) when the vector field magnitude is zero; requires evaluation of Gaussian gradients which may be computationally intensive for large $N$.
*   **When to use**: When working with Oriented Gaussian Splatting representations and needing explicit surface normals for mesh extraction or training constraints [S1].
*   **When not to use**: When high-performance real-time particle advection is needed without the overhead of summing $N$ gradients; when the wrapping assumption does not hold.

## Background
The Gaussian Normal Field Definition originates from the paper "From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians" [S1]. It is presented as Definition 3 and Proposition 3 in this work. The method leverages results from "Objects as Volumes" (OaV) [S1, ref 39] regarding reciprocal exponential transport and the gradient of the log-vacancy. It is used in the context of 3D Gaussian Splatting for surface reconstruction, specifically to supervise oriented Gaussians and extract meshes [S1].

## Sources
[S1] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
