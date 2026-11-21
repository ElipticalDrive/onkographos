# Hessian Estimation via Convolution

Also called: Hessian Estimation (Our Hess.), Second-order derivative estimation using the Hessian operator

## Overview
Hessian Estimation via Convolution computes the second-order spatial derivatives (the Hessian matrix) of a scalar volume function by convolving the volume with a kernel that combines Gaussian smoothing and second-order differentiation. It solves the problem of obtaining stable, differentiable curvature information required for higher-order optimization in differentiable volume rendering. It sets itself apart from [Smooth Gradient Estimation via Convolution](6449-smooth-gradient-estimation-via-convolution.md) by operating on the second derivative operator $\nabla^2$ rather than the first, and from [Gradient Estimation (Our Grad.)](6476-gradient-estimation-our-grad.md) by producing an $n \times n$ matrix field instead of a vector field.

## How it works
**Inputs:** A scalar volume function $V(\mathbf{x})$ defined on $\mathbb{R}^n$; a smoothing parameter $\sigma > 0$.
**Outputs:** An $n \times n$ Hessian field $H(\mathbf{x})$, where $H_{ij} = \frac{\partial^2 V}{\partial x_i \partial x_j}$.

**Data Structures:**
*   **Volume Grid:** $V$ stored as a dense array or sparse structure.
*   **Hessian Field:** $H$ stored as an $n \times n$ matrix per voxel. Due to symmetry ($H_{ij} = H_{ji}$), only the upper triangular part (including diagonal) needs storage, reducing memory by half [S1].
*   **CDF Lookup Table:** For diagonal elements, a 1D table mapping CDF values to spatial offsets $\tau_i$.

**Method Steps:**

1.  **Define the Kernel:** The convolution kernel $D^H \kappa_{i,j}(\boldsymbol{\tau})$ combines the Gaussian $N(\boldsymbol{\tau}, \sigma)$ with the second-order differential operator. For dimensions $i, j \in \{1, \dots, n\}$:
    $$
    D^H \kappa_{i,j}(\boldsymbol{\tau}) =
    \begin{cases}
    \left( -\frac{\tau_i^2}{\sigma^2} + \frac{1}{\sigma^2} \right) N(\boldsymbol{\tau}, \sigma) & \text{if } i = j \\
    \frac{\tau_i \tau_j}{\sigma^4} N(\boldsymbol{\tau}, \sigma) & \text{if } i \neq j
    \end{cases}
    $$
    where $N(\boldsymbol{\tau}, \sigma) = \frac{1}{(2\pi\sigma^2)^{n/2}} \exp\left(-\frac{\|\boldsymbol{\tau}\|^2}{2\sigma^2}\right)$ [S1].

2.  **Sampling Strategy (Stochastic Estimation):** To estimate the Hessian efficiently, samples are drawn from a probability density function (PDF) derived from the kernel.
    *   **Off-Diagonal Elements ($i \neq j$):** The kernel factorizes into the product of two first-order gradient kernels. Sample $\tau_i$ from the PDF of the first derivative in dimension $i$, and $\tau_j$ from the PDF of the first derivative in dimension $j$. The estimate is the product of these samples scaled appropriately [S1].
    *   **Diagonal Elements ($i = j$):** The kernel is signed and does not form a valid PDF directly. It must be positivized and normalized. The Cumulative Distribution Function (CDF) $P^H_{ii}(u)$ for the marginal distribution along dimension $i$ is:
        $$
        P^H_{ii}(u) =
        \begin{cases}
        \frac{1}{4\sigma} \exp\left(-\frac{u^2}{2\sigma^2}\right) & \text{if } u < -\sigma \\
        0.5 + \frac{u}{4\sigma} - \frac{1}{4\sigma} \exp\left(-\frac{u^2}{2\sigma^2}\right) & \text{if } u \in [-\sigma, \sigma] \\
        1 - \frac{1}{4\sigma} \exp\left(-\frac{u^2}{2\sigma^2}\right) & \text{if } u > \sigma
        \end{cases}
        $$
        This is a transcendental equation with no closed-form inverse [S1].

3.  **Inverse CDF Implementation:**
    *   Tabulate $P^H_{ii}(u)$ on a discrete range $[-10\sigma, 10\sigma]$ for each dimension $i$.
    *   Pre-compute and store the inverse indices (mapping CDF values to $u$) in a sorted array to allow $O(1)$ lookup during sampling [S1].

4.  **Estimation:**
    *   For each sample, compute the contribution to the Hessian matrix.
    *   Exploit symmetry: Estimate $H_{ij}$ and $H_{ji}$ simultaneously.
    *   Average contributions over $M$ samples to reduce variance.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def gaussian_pdf(tau, sigma):
    """Standard multivariate Gaussian PDF."""
    norm_factor = 1.0 / (math.sqrt(2 * math.pi) * sigma)
    exp_term = math.exp(-0.5 * (tau * tau) / (sigma * sigma))
    return norm_factor * exp_term

def compute_diagonal_cdf(u, sigma):
    """
    Computes the CDF P^H_ii(u) for diagonal Hessian elements.
    """
    if u < -sigma:
        return (1.0 / (4 * sigma)) * math.exp(-0.5 * (u * u) / (sigma * sigma))
    elif u <= sigma:
        return 0.5 + (u / (4 * sigma)) - (1.0 / (4 * sigma)) * math.exp(-0.5 * (u * u) / (sigma * sigma))
    else:
        return 1.0 - (1.0 / (4 * sigma)) * math.exp(-0.5 * (u * u) / (sigma * sigma))

def sample_diagonal_element(cdf_table, inverse_indices, sigma):
    """
    Samples tau_i from the diagonal Hessian distribution using inverse transform sampling.
    cdf_table: Pre-computed array of CDF values.
    inverse_indices: Pre-computed array mapping CDF index to tau value.
    """
    # Generate uniform random number in [0, 1]
    r = random.uniform(0.0, 1.0)
    # Find index in CDF table
    idx = binary_search(cdf_table, r)
    # Return corresponding tau value
    return inverse_indices[idx]

def estimate_hessian_volume(volume, sigma, num_samples):
    """
    Estimates the Hessian field of a volume using stochastic convolution.
    """
    n = volume.dimensions
    hessian_field = zeros(n, n, volume.shape)
    
    # Pre-compute diagonal CDF tables for each dimension
    # Range [-10*sigma, 10*sigma]
    tau_range = linspace(-10*sigma, 10*sigma, 1000)
    cdf_tables = {}
    inverse_indices = {}
    
    for i in range(n):
        cdf_vals = [compute_diagonal_cdf(t, sigma) for t in tau_range]
        # Sort CDF values and store corresponding tau indices for O(1) access
        sorted_indices = argsort(cdf_vals)
        cdf_tables[i] = [cdf_vals[k] for k in sorted_indices]
        inverse_indices[i] = [tau_range[k] for k in sorted_indices]

    for voxel in volume.voxels:
        hessian = zeros(n, n)
        
        for _ in range(num_samples):
            # Sample off-diagonal and diagonal elements
            # Note: In practice, one might sample all n^2 elements or use importance sampling
            
            # Diagonal elements
            for i in range(n):
                tau_i = sample_diagonal_element(cdf_tables[i], inverse_indices[i], sigma)
                # Kernel value for diagonal: (-tau_i^2/sigma^2 + 1/sigma^2) * N(tau_i, sigma)
                # N(tau_i, sigma) is 1D Gaussian for this dimension
                # Other dimensions are integrated out or sampled as standard Gaussians
                # For simplicity, assuming we sample the full vector tau
                # The contribution is V(voxel + tau) * Kernel(tau)
                
                # Construct full tau vector: tau_i is fixed, others sampled from N(0, sigma)
                tau = zeros(n)
                tau[i] = tau_i
                for j in range(n):
                    if j != i:
                        tau[j] = random.gauss(0, sigma)
                
                # Compute kernel weight
                # D^H kappa_{ii} = (-tau_i^2/sigma^2 + 1/sigma^2) * N(tau, sigma)
                # N(tau, sigma) = Product of 1D Gaussians
                n_val = 1.0
                for k in range(n):
                    n_val *= gaussian_pdf(tau[k], sigma)
                
                kernel_val = (-tau[i]**2 / sigma**2 + 1.0 / sigma**2) * n_val
                
                # Add contribution
                # In stochastic estimation, we evaluate V at voxel + tau
                v_val = volume.evaluate(voxel + tau)
                hessian[i, i] += v_val * kernel_val

            # Off-diagonal elements
            for i in range(n):
                for j in range(i + 1, n):
                    # Sample tau_i and tau_j independently from first-order gradient distributions
                    # First-order gradient PDF for dim k: p(tau_k) proportional to |tau_k| * N(tau_k, sigma)
                    # However, the source states off-diagonals are product of partial gradients.
                    # We sample tau_i and tau_j from the respective 1D gradient PDFs.
                    tau_i = sample_gradient_pdf(i, sigma) # Helper for first-order gradient sampling
                    tau_j = sample_gradient_pdf(j, sigma)
                    
                    tau = zeros(n)
                    tau[i] = tau_i
                    tau[j] = tau_j
                    for k in range(n):
                        if k != i and k != j:
                            tau[k] = random.gauss(0, sigma)
                    
                    # Kernel value: (tau_i * tau_j / sigma^4) * N(tau, sigma)
                    n_val = 1.0
                    for k in range(n):
                        n_val *= gaussian_pdf(tau[k], sigma)
                    
                    kernel_val = (tau[i] * tau[j] / sigma**4) * n_val
                    
                    v_val = volume.evaluate(voxel + tau)
                    hessian[i, j] += v_val * kernel_val
                    hessian[j, i] = hessian[i, j] # Symmetry

        # Average over samples
        hessian /= num_samples
        hessian_field[voxel] = hessian
        
    return hessian_field
```

**Edge Cases and Pitfalls:**
*   **Precision:** The diagonal CDF is transcendental; numerical instability may occur near $u = \pm \sigma$ if not handled with high-precision arithmetic or careful tabulation [S1].
*   **Boundary Conditions:** Sampling near volume boundaries requires handling out-of-bounds accesses (e.g., clamping, reflection, or zero-padding) [G].
*   **Variance:** Stochastic estimation has high variance; [Aggregate Importance Sampling](6471-aggregate-importance-sampling.md) is recommended to reduce this [S1].

## Subtopics

### [Aggregate Importance Sampling](6471-aggregate-importance-sampling.md)
A variance reduction technique that samples from a mixture distribution of all convolution kernels, reducing computational complexity from $O(n^2)$ to $O(1)$ per iteration at the cost of slightly increased variance.

### [Second Directional Derivative](11718-second-directional-derivative.md)
Uses the Hessian to compute the second directional derivative along the gradient direction, providing sharper edge characterization than gradient magnitude alone.

### [Edge location estimation via second-order derivative](15889-edge-location-estimation-via-second-order-derivative.md)
Locates step edges by finding the zero-crossing of the second-order derivative, relying on the continuity of the reconstruction filter.

### Separable Hessian and Gradient Computation
A deterministic method that exploits the separability of Gaussian derivative kernels to compute the Hessian matrix using sequential 1D convolutions. It reduces the complexity of a 3D convolution $O(k^3)$ to three 1D convolutions $O(3k)$. Computing the gradient requires 8 1D convolutions (reusing $G_\sigma(x)*I$ twice and $G'_\sigma(x)*I$ once), while computing the Hessian requires 15 1D convolutions (reusing $G_\sigma(x)*I$ three times and $G'_\sigma(x)*I$ twice) [S2]. The formula for the derivative is $\partial^o I = \partial_x^a \partial_y^b \partial_z^c I = G_\sigma^{(a)} * G_\sigma^{(b)} * G_\sigma^{(c)} * I$ where $a+b+c=o \in \{1, 2\}$ [S2]. Benchmarks on a Pentium III 800 MHz with kernel size $k=7$ show that computing Hessian eigenvalues is on average 2.7 times more expensive than computing gradient magnitude [S2].

## Operating characteristics
*   **Time Complexity:** $O(n^2)$ function evaluations per voxel without Importance Sampling (IS). With IS, it remains $O(n^2)$ but with lower variance. With Aggregate IS, it reduces to $O(1)$ per iteration [S1].
*   **Memory Complexity:** $O(n^2)$ per voxel for the Hessian matrix, though symmetry allows storing only $n(n+1)/2$ elements [S1].
*   **Performance:** Deterministic separable convolution is significantly faster than stochastic sampling for small kernels but scales poorly with kernel size ($O(k^3)$ vs $O(k)$ for separable) [S2].
*   **Strengths:** Provides full curvature information; stable via Gaussian smoothing; compatible with gradient-based optimization.
*   **Limitations:** High computational cost for full Hessian; stochastic methods suffer from noise without sufficient samples or IS; deterministic methods are memory-intensive for large volumes [S1, S2].
*   **When to use:** When second-order information is critical for convergence or edge detection. Avoid when memory is constrained or real-time performance is required without IS.

## Background
The method originates from stochastic gradient estimation for differentiable rendering, specifically extending first-order gradient estimation to second-order derivatives. The core formulation for the Hessian kernel and its stochastic sampling via CDF inversion was introduced by Fischer and Ritschel in "Stochastic Gradient Estimation for Higher-Order Differentiable Rendering" (2025) [S1]. The separable convolution approach for Hessian computation in volume data was earlier established by Weinkauf et al. in "Salient Representation of Volume Data" (2001) [S2].

## Sources
[S1] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
[S2] Salient Representation of Volume Data, https://doi.org/10.2312/vissym/vissym01/203-212, fetched 2026-10-30
