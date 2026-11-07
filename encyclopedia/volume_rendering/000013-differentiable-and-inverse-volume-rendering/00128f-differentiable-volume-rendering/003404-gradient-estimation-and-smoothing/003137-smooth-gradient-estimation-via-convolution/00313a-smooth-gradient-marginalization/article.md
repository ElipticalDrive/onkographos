# Smooth Gradient Marginalization

Also called: Marginalization of Gaussian Gradient

## Overview
Smooth Gradient Marginalization is a technique to sample an $n$-dimensional Gaussian gradient by marginalizing the probability density function (PDF) over all dimensions except the one being differentiated, allowing individual dimension sampling [S1]. It differs from methods that only blur in the direction of differentiation by blurring all dimensions, ensuring consistency with higher-order differentials [S1].

## How it works
**Inputs:** An $n$-dimensional Gaussian function $N(\tau)$ with standard deviation $\sigma$, where $\tau$ is a vector.
**Outputs:** The marginal PDF $p_G(\tau)$ for a specific dimension $i$, and the components of the Hessian matrix $\nabla^2 N(\tau)$.

**Data Structures:**
*   $\tau$: Vector of size $n$.
*   $\sigma$: Scalar standard deviation.
*   $N(\tau)$: The $n$-dimensional Gaussian value.
*   $N(\tau_i)$: The 1-dimensional Gaussian value for the $i$-th component.

**Steps:**
1.  **Marginalization:** To sample the gradient for dimension $i$, integrate the $n$-dimensional Gaussian gradient PDF $p_G(\tau)$ over all dimensions $j \neq i$ [S1]:
    $$ p_G(\tau) = \int p_G(\tau) d\tau_{j \neq i} $$
    [S1]

2.  **Derivation of Marginal PDF:** The $n$-dimensional Gaussian is the product of $n$ 1D Gaussians. When differentiating with respect to $\tau_i$, the factors for $j \neq i$ integrate to 1 (the domain's measure) [S1]. The marginal PDF is proportional to the absolute value of the 1D gradient:
    $$ p_G(\tau) \propto |\nabla_i N(\tau_i)| $$
    [S1]

3.  **Hessian Diagonal Calculation:** The diagonal element of the Hessian is the second derivative with respect to $\tau_i$. It is **not** a special case of the off-diagonal element because the variable appears twice in the differentiation ($d(u^2)/du = 2u$ vs $d(uv)/du = v$) [S1]. The formula is:
    $$ \nabla^2 N(\tau)_{ii} = \frac{\partial^2 N(\tau)}{\partial \tau_i^2} = \left(\frac{\tau_i^2}{\sigma^4} - \frac{1}{\sigma^2}\right) N(\tau) $$
    [S1]

4.  **Hessian Off-Diagonal Calculation:** For $i \neq j$, the off-diagonal element is:
    $$ \nabla^2 N(\tau)_{ij} = \frac{\tau_i \tau_j}{\sigma^4} N(\tau) $$
    [S1]

**Pitfalls:**
*   **Diagonal vs. Off-Diagonal:** Do not substitute $i=j$ into the off-diagonal formula. The diagonal requires the second derivative of the same variable, yielding a different coefficient structure [S1].
*   **Dimension Blurring:** Unlike [Fischer and Ritschel], which only blurs in the differentiation direction, this method blurs all dimensions to maintain consistency for higher-order differentials [S1].

## Subtopics

### [Sampling Diagonal of Hessian](6477-sampling-diagonal-of-hessian.md)
Derives the valid distribution for the second-order derivative of the 1D Gaussian by positivization and scaling, applicable to higher dimensions.

## Operating characteristics
Not given in the sources.

## Background
The sampling of 1D Gaussian gradients was originally derived by Fischer and Ritschel [7] [S1]. This method extends that work to $n$-dimensions by marginalizing the PDF, addressing the need for consistent higher-order differentials in stochastic gradient estimation [S1].

## Sources
[S1] Stochastic Gradient Estimation for Higher-Order Differentiable Rendering, https://doi.org/10.1109/iccv51701.2025.02618, fetched 2026-10-29
