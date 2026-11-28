# Hessian Estimation via Convolution

Also called: Hessian Estimation (Our Hess.), Second-order derivative estimation using the Hessian operator

## Overview
Hessian Estimation via Convolution computes the second-order spatial derivatives (the Hessian matrix) of a scalar volume function by convolving the volume with a kernel that combines Gaussian smoothing and second-order differentiation [S1]. It solves the problem of obtaining stable, differentiable curvature information required for higher-order optimization in differentiable volume rendering [G]. It sets itself apart from [Smooth Gradient Estimation via Convolution](6449-smooth-gradient-estimation-via-convolution.md) by operating on the second derivative operator $\nabla^2$ rather than the first, and from [Gradient Estimation (Our Grad.)](6476-gradient-estimation-our-grad.md) by producing an $n \times n$ matrix field instead of a vector field [S1].

## How it works
**Inputs:** A scalar volume function $V(\mathbf{x})$ defined on $\mathbb{R}^n$; a smoothing parameter $\sigma > 0$ [S1].
**Outputs:** An $n \times n$ Hessian field $H(\mathbf{x})$, where $H_{ij} = \frac{\partial^2 V}{\partial x_i \partial x_j}$ [S1].

**Data Structures:**
*   **Volume Grid:** $V$ stored as a dense array or sparse structure [G].
*   **Hessian Field:** $H$ stored as an $n \times n$ matrix per voxel. The method exploits symmetry ($H_{ij} = H_{ji}$) during sampling and estimation to reduce computational overhead [S1].
*   **CDF Lookup Table:** For diagonal elements, a 1D table mapping CDF values to spatial offsets $\tau_i$ [S1].

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

2.  **Sampling Strategy (Stochastic Estimation):** To estimate the Hessian efficiently, samples are drawn from a probability density function (PDF) derived from the kernel. The kernel is positivized and normalized to form a valid distribution [S1].
    *   **Off-Diagonal Elements ($i \neq j$):** The kernel factorizes into the product of two first-order gradient kernels. Sample $\tau_i$ from the PDF of the first derivative in dimension $i$, and $\tau_j$ from the PDF of the first derivative in dimension $j$ [S1].
    *   **Diagonal Elements ($i = j$):** The kernel is signed and must be positivized. The Cumulative Distribution Function (CDF) $P^H_{ii}(u)$ for the marginal distribution along dimension $i$ is:
        $$
        P^H_{ii}(u) =
        \begin{cases}
        \frac{1}{4\sigma} \exp\left(-\frac{u^2}{2\sigma^2}\right) & \text{if } u < -\sigma \\
        0.5 + \frac{u}{4\sigma} \exp\left(-\frac{u^2}{2\sigma^2}\right) & \text{if } u \in [-\sigma, \sigma] \\
        1 - \frac{1}{4\sigma} \exp\left(-\frac{u^2}{2\sigma^2}\right) & \text{if } u > \sigma
        \end{cases}
        $$
        This is a transcendental equation with no closed-form inverse [S1].

3.  **Inverse CDF Implementation:**
    *   Tabulate $P^H_{ii}(u)$ on a discrete range $[-10\sigma, 10\sigma]$ for each dimension $i$ [S1].
    *   Pre-compute and store the inverse indices (mapping CDF values to $u$) in a sorted array to allow $O(1)$ lookup during sampling [S1].

4.  **Estimation:**
    *   For each sample, compute the contribution to the Hessian matrix by evaluating the volume at the sampled offset and multiplying by the kernel weight [S1].
    *   Exploit symmetry: Estimate $H_{ij}$ and $H_{ji}$ simultaneously [S1].
    *   Average contributions over $M$ samples to reduce variance [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def gaussian_pdf_1d(tau, sigma):
    """1D Gaussian PDF."""
    return (1.0 / (math.sqrt(2 * math.pi) * sigma)) * math.exp(-0.5 * (tau * tau) / (sigma * sigma))

def compute_diagonal_cdf(u, sigma):
    """CDF P^H_ii(u) for diagonal Hessian elements."""
    if u < -sigma:
        return (1.0 / (4 * sigma)) * math.exp(-0.5 * (u * u) / (sigma * sigma))
    elif u <= sigma:
        return 0.5 + (u / (4 * sigma)) * math.exp(-0.5 * (u * u) / (sigma * sigma))
    else:
        return 1.0 - (1.0 / (4 * sigma)) * math.exp(-0.5 * (u * u) / (sigma * sigma))

def binary_search(arr, target):
    """Standard binary search returning the index of target in sorted arr."""
    low, high = 0, len(arr) - 1
    while low <= high:
        mid = (low + high) // 2
        if arr[mid] < target:
            low = mid + 1
        else:
            high = mid - 1
    return low

def sample_diagonal_element(cdf_table, inverse_indices):
    """Samples tau_i from the diagonal Hessian distribution using inverse transform sampling."""
    r = math.random()
    idx = binary_search(cdf_table, r)
    return inverse_indices[idx]

def sample_gradient_pdf_1d(sigma):
    """Samples from the 1D first-order gradient PDF p(tau) ~ |tau| * exp(-tau^2 / 2sigma^2) via rejection sampling."""
    while True:
        tau = math.random() * 2 * sigma - sigma
        if tau == 0.0:
            tau = 1e-8
        u = math.random()
        # Rejection threshold: |tau| * exp(-tau^2 / 2sigma^2) / max_val
        # max_val occurs at tau = sigma, value = sigma * exp(-0.5)
        max_val = sigma * math.exp(-0.5)
        if u <= (abs(tau) * math.exp(-0.5 * (tau * tau) / (sigma * sigma))) / max_val:
            return tau

def estimate_hessian_volume(volume, sigma, num_samples):
    """Estimates the Hessian field of a volume using stochastic convolution."""
    n = volume.dimensions
    hessian_field = [[0.0] * n for _ in range(volume.shape[0])]
    
    # Pre-compute diagonal CDF tables for each dimension
    tau_range = [(-10 * sigma) + i * (20 * sigma / 999) for i in range(1000)]
    cdf_tables = {}
    inverse_indices = {}
    
    for i in range(n):
        cdf_vals = [compute_diagonal_cdf(t, sigma) for t in tau_range]
        # Sort CDF values and store corresponding tau indices for O(1) access
        sorted_indices = sorted(range(len(cdf_vals)), key=lambda k: cdf_vals[k])
        cdf_tables[i] = [cdf_vals[k] for k in sorted_indices]
        inverse_indices[i] = [tau_range[k] for k in sorted_indices]

    for voxel_idx in range(volume.shape[0]):
        hessian = [[0.0] * n for _ in range(n)]
        
        for _ in range(num_samples):
            # Diagonal elements
            for i in range(n):
                tau_i = sample_diagonal_element(cdf_tables[i], inverse_indices[i])
                # Construct full tau vector: tau_i is fixed, others sampled from N(0, sigma)
                tau = [0.0] * n
                tau[i] = tau_i
                for j in range(n):
                    if j != i:
                        # Box-Muller transform for standard normal
                        u1 = math.random()
                        u2 = math.random()
                        tau[j] = sigma * math.sqrt(-2.0 * math.log(u1)) * math.cos(2.0 * math.pi * u2)
                
                # Compute kernel weight: D^H kappa_{ii} = (-tau_i^2/sigma^2 + 1/sigma^2) * N(tau, sigma)
                n_val = 1.0
                for k in range(n):
                    n_val *= gaussian_pdf_1d(tau[k], sigma)
                kernel_val = (-tau[i]**2 / sigma**2 + 1.0 / sigma**2) * n_val
                
                # Monte Carlo estimator: V(x + tau) * Kernel(tau)
                v_val = volume.evaluate(voxel_idx, tau)
                hessian[i][i] += v_val * kernel_val

            # Off-diagonal elements
            for i in range(n):
                for j in range(i + 1, n):
                    tau_i = sample_gradient_pdf_1d(sigma)
                    tau_j = sample_gradient_pdf_1d(sigma)
                    
                    tau = [0.0] * n
                    tau[i] = tau_i
                    tau[j] = tau_j
                    for k in range(n):
                        if k != i and k != j:
                            u1 = math.random()
                            u2 = math.random()
                            tau[k] = sigma * math.sqrt(-2.0 * math.log(u1)) * math.cos(2.0 * math.pi * u2)
                    
                    # Kernel value: (tau_i * tau_j / sigma^4) * N(tau, sigma)
                    n_val = 1.0
                    for k in range(n):
                        n_val *= gaussian_pdf_1d(tau[k], sigma)
                    kernel_val = (tau[i] * tau[j] / sigma**4) * n_val
                    
                    v_val = volume.evaluate(voxel_idx, tau)
                    hessian[i][j] += v_val * kernel_val
                    hessian[j][i] = hessian[i][j] # Symmetry

        # Average over samples
        for i in range(n):
            for j in range(n):
                hessian[i][j] /= num_samples
        hessian_field[voxel_idx] = hessian
        
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
A deterministic method that exploits the separability of Gaussian derivative kernels to compute the Hessian matrix using sequential 1D convolutions. It reduces the complexity of a 3D convolution $O(k^3)$ to three 1D convolutions $O(3k)$ [S2]. Computing the gradient requires 8 1D convolutions (reusing $G_\sigma(x)*I$ twice and $G'_\sigma(x)*I$ once), while computing the Hessian requires 15 1D convolutions (reusing $G_\sigma(x)*I$ three times and $G'_\sigma(x)*I$ twice) [S2]. The formula for the derivative is $\partial^o I = \partial_x^a \partial_y^b \partial_z^c I = G_\sigma^{(a)} * G_\sigma^{(b)} * G_\sigma^{(c)} * I$ where $a+b+c=o \in \{1, 2\}$ [S2]. Benchmarks on a Pentium III 800 MHz with kernel size $k=7$ show that computing Hessian eigenvalues is on average 2.7 times more expensive than computing gradient magnitude [S2].

## Operating characteristics
*   **Time Complexity:** $O(n^2)$ function evaluations per voxel without Importance Sampling (IS). With IS, it remains $O(n^2)$ but with lower variance. With Aggregate IS, it reduces to $O(1)$ per iteration [S1].
*   **Memory Complexity:** $O(n^2)$ per voxel for the full Hessian matrix, though symmetry allows storing only $n(n+1)/2$ elements [S1]. Avoiding the full Hessian is possible by using Hessian-vector products (HVPs) [S1].
*   **Performance:** Deterministic separable convolution reduces 3D convolution complexity $O(k^3)$ to three 1D convolutions $O(3k)$ [S2].
*   **Strengths:** Provides full curvature information; stable via Gaussian smoothing; compatible with gradient-based optimization [S1, S2].
*   **Limitations:** High computational cost for full Hessian; stochastic methods suffer from noise without sufficient samples or IS; deterministic methods are memory-intensive for large volumes [S1, S2].
*   **When to use:** When second-order information is critical for convergence or edge detection. Avoid when memory is constrained or real-time performance is required without IS [S1, S2].

## Background
The method originates from stochastic gradient estimation for differentiable rendering, specifically extending first-order gradient estimation to second-order derivatives. The core formulation for the Hessian kernel and its stochastic sampling via CDF inversion was introduced by Fischer and Ritschel in "Stochastic Gradient Estimation for Higher-Order Differentiable Rendering" (2025) [S1]. The separable convolution approach for Hessian computation in volume data was earlier established by Weinkauf et al. in "Salient Representation of Volume Data" (2001) [S2].

## Sources
[S1] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
[S2] Salient Representation of Volume Data, https://doi.org/10.2312/vissym/vissym01/203-212, fetched 2026-10-30
