# Particle-Wise Higher-Order SPH Field Approximation for DVR

Also called: Higher-order SPH field approximation, Particle-Wise Higher-Order SPH Field Approximation, Piecewise polynomial SPH approximation

## Overview
This method approximates Smoothed Particle Hydrodynamics (SPH) scalar fields along viewing rays using piecewise polynomial functions of higher order, addressing the challenge of rendering SPH data which has highly varying spatial resolution [S1]. It sets itself apart from linear or constant approximations by minimizing the $L^2$ distance between the approximation and the exact particle kernel contribution for each particle individually, allowing the representation resolution to adapt to the data resolution [S1]. Unlike sibling methods that focus on sampling strategies or entropy-based selection, this technique focuses on the mathematical fidelity of the field representation along the ray [S1, 3626-importance-sampling-for-sph-rendering.md, 3638-stochastic-volume-rendering-of-multi-phase-sph-data.md].

## How it works
The method consists of three main passes: (1) approximating each particle's contribution to relevant viewing rays, (2) sorting contributions per ray, and (3) accumulating and compositing [S1].

### Inputs and Outputs
**Inputs:**
*   SPH particle dataset: positions $\chi$, mass $\mu$, target field value $\alpha$, density $\rho$, smoothing radius $\zeta$.
*   Viewing rays defined by base point $b$ and unit direction vector $v$.
*   Approximation parameters: maximum polynomial degree $D$ and number of non-trivial pieces $K$.
*   SPH kernel function $w(r)$ (typically cubic B-spline).

**Outputs:**
*   Rendered image with scalar field values.

### Data Structures
**1. Look-up Table (LUT) for Optimal Approximations:**
*   Precomputed for equidistant values of $\Lambda$ (normalized distance from particle to ray).
*   Stores localized difference coefficients $\hat{a}_{kd}$ for the optimal approximation of the unit particle contribution $B_\Lambda$.
*   Structure: Array indexed by $\Lambda$, containing sequences of knots $(t_k, \hat{a}_{k0}, \dots, \hat{a}_{kD})$.

**2. Ray Contribution List:**
*   For each ray, a list of contributions from particles.
*   Each contribution is encoded as a sequence of knots: $(t_k, \hat{a}_{k0}, \dots, \hat{a}_{kD})$.
*   $t_k$: Ray parameter (knot position).
*   $\hat{a}_{kd}$: Localized difference coefficients representing the change in polynomial behavior at $t_k$.

### Mathematical Foundation

**SPH Kernel:**
The cubic B-spline kernel is used [S1]:
$$w(r) = \frac{1}{4\pi} \begin{cases} (2-r)^3 - 4(1-r)^3, & 0 \leq r < 1 \\ (2-r)^3, & 1 \leq r < 2 \\ 0, & 2 \leq r \end{cases}$$

**Particle Contribution:**
A particle's contribution to a scalar field at position $x$ is [S1]:
$$\frac{\mu\alpha}{\rho\zeta^3} w\left(\frac{\|x - \chi\|}{\zeta}\right)$$

**Normalized Particle Contribution on a Ray:**
For a ray $x(t) = b + tv$, let $t_\chi$ be the parameter of the point on the ray closest to particle position $\chi$. Let $\Lambda = \frac{1}{\zeta} \|\chi - x(t_\chi)\|$. The normalized contribution (unit mass, density, field attribute, smoothing radius) is [S1]:
$$B_\Lambda(t) = w\left(\sqrt{\Lambda^2 + t^2}\right)$$
The actual contribution of a data particle is [S1]:
$$\frac{\mu\alpha}{\rho\zeta^3} B_\Lambda\left(\frac{t - t_\chi}{\zeta}\right)$$

**Polynomial Representation:**
A polynomial $A(t)$ of order $D$ is represented by coefficients $a_d$:
$$A(t) = \sum_{d=0}^D a_d t^d$$
Localized coefficients $a_d$ at $t_*$ are computed from standard coefficients $a_j$ as [S1]:
$$a_{d} = \sum_{j=d}^D \binom{j}{d} a_j t_*^{j-d}$$
Such that $A(t) = \sum_{d=0}^D a_{d} (t - t_*)^d$.

**Piecewise Polynomial Encoding:**
A continuous piecewise polynomial function is defined by knots $t_k$ and localized difference coefficients $\hat{a}_{kd}$ such that the difference between polynomials $A_k$ (right of $t_k$) and $A_{k-1}$ (left of $t_k$) is [S1]:
$$A_k(t) - A_{k-1}(t) = \sum_{d=0}^D \hat{a}_{kd} (t - t_k)^d$$
Summing piecewise polynomials reduces to sorting their joint knots. The localized coefficients $a_{kd}$ for piece $k$ are updated from piece $k-1$ using [S1]:
$$a_{kd} = \hat{a}_{kd} + \sum_{j=d}^D \binom{j}{d} a_{(k-1)j} (t_k - t_{k-1})^{j-d} \quad \text{for } d = 0, \dots, D$$

**Optimal Approximation:**
The goal is to minimize the $L^2$ distance error [S1]:
$$E_\Lambda(S) = \|S - B_\Lambda\|_2 = \left( \int_{\mathbb{R}} [S(t) - B_\Lambda(t)]^2 dt \right)^{\frac{1}{2}}$$
The space $\mathcal{S}$ consists of even, continuous, piecewise polynomial functions with compact support, maximal degree $D$, and $K$ non-trivial pieces. The dimension of $\mathcal{S}$ is $\lfloor \frac{KD}{2} \rfloor$.

**Basis Construction:**
1.  Define non-orthogonal basis $\tilde{\mathcal{A}} = \{\tilde{A}_{kd}\}$ where $\mathcal{J}$ is the set of pairs $(k, d)$ with $k \leq \lceil K/2 \rceil$ and $d \leq D$, excluding $(1, d)$ for uneven $d$ if $K$ is uneven [S1].
2.  Basis functions are [S1]:
    $$\tilde{A}_{kd}(t) = \begin{cases} 1 & \text{if } |t| \leq \theta_{k-1} \\ 1 - \left( \frac{|t| - \theta_{k-1}}{\theta_k - \theta_{k-1}} \right)^d & \text{if } \theta_{k-1} \leq |t| \leq \theta_k \\ 0 & \text{if } |t| \geq \theta_k \end{cases}$$
    where $\theta_0 = 0$ and $\theta_k$ are positive knot positions.
3.  Convert to orthogonal basis $\mathcal{A}$ using Gram-Schmidt in lexicographical order [S1]:
    $$A_{k^*d^*} = \tilde{A}_{d^*k^*} - \sum_{\substack{(k,d) \in \mathcal{J} \\ k < k^* \vee (k = k \wedge d < d^*)}} \text{proj}_{A_{kd}}(\tilde{A}_{d^*k^*})$$
    where $\text{proj}_B(A) = \frac{\langle A, B \rangle}{\langle B, B \rangle} B$ and $\langle A, B \rangle = \int_{\mathbb{R}} A(t) B(t) dt$.

**Optimal Knot Positions:**
Find $\theta_1, \dots, \theta_{\lceil K/2 \rceil}$ by minimizing $E_\Lambda$ via non-linear optimization over $0 < \theta_1 < \dots < \theta_{\lceil K/2 \rceil}$ [S1]. The objective function $E_\Lambda$ is computed using closed-form symbolic formulas for inner products $\langle A, B_\Lambda \rangle$, generated once per kernel type using computer algebra systems. For $\Lambda$ close to the upper bound, multiple precision arithmetic (e.g., GNU MPFR) is required for stability [S1].

The optimal approximation $S_\Lambda$ for fixed knots is the orthogonal projection of $B_\Lambda$ onto $\mathcal{S}$ [S1]:
$$S_\Lambda = \sum_{A \in \mathcal{A}} \text{proj}_A(B_\Lambda)$$

### Algorithm Steps

1.  **Precomputation:**
    *   For the chosen kernel (e.g., cubic B-spline), generate closed-form symbolic formulas for inner products $\langle A, B_\Lambda \rangle$ for variable knot positions $\theta_k$ and distance $\Lambda$ [S1].
    *   For a range of equidistant $\Lambda$ values, perform non-linear optimization to find optimal knot positions $\theta_k$ that minimize $E_\Lambda$ [S1].
    *   For each $\Lambda$ and its optimal knots, compute the orthogonal basis $\mathcal{A}$ and the optimal approximation $S_\Lambda$ [S1].
    *   Encode $S_\Lambda$ as a sequence of knots $(t_k, \hat{a}_{k0}, \dots, \hat{a}_{kD})$ and store in LUT [S1].

2.  **Pass 1: Particle Approximation:**
    *   For each particle $i$ with parameters $\mu_i, \alpha_i, \rho_i, \zeta_i, \chi_i$:
        *   For each viewing ray $j$:
            *   Compute $t_{\chi, ij}$ (closest point parameter) and $\Lambda_{ij} = \frac{1}{\zeta_i} \|\chi_i - x_j(t_{\chi, ij})\|$ [S1].
            *   If $\Lambda_{ij}$ is within kernel support:
                *   Retrieve precomputed localized difference coefficients $\hat{a}_{kd}$ from LUT for $\Lambda_{ij}$ [S1].
                *   Scale and translate coefficients to match particle parameters and ray position [S1]. The contribution is $\frac{\mu_i\alpha_i}{\rho_i\zeta_i^3} B_{\Lambda_{ij}}\left(\frac{t - t_{\chi, ij}}{\zeta_i}\right)$. The encoding in terms of knots $(t_k, \hat{a}_{kd})$ must be adjusted for scaling by $\zeta_i$ and translation by $t_{\chi, ij}$, and amplitude scaling by $\frac{\mu_i\alpha_i}{\rho_i\zeta_i^3}$. [G: The scaling of knots and coefficients for arbitrary $\zeta$ and $t_\chi$ requires applying the change of variables to the polynomial pieces. Specifically, if the unit approximation is defined on $[t_k, t_{k+1}]$, the scaled approximation is on $[\zeta t_k + t_\chi, \zeta t_{k+1} + t_\chi]$. The coefficients $\hat{a}_{kd}$ must be transformed accordingly. The source states "by just translating and scaling it in the same way", implying the LUT provides the base shape, and the rendering pass applies the affine transformation to the ray parameter and scales the amplitude. The exact transformation of $\hat{a}_{kd}$ under scaling $t \to \zeta t + t_\chi$ is not explicitly detailed in the notes, but is a standard polynomial transformation. For implementation, one would compute the new coefficients for the scaled polynomial.]
                *   Add the encoded contribution (knots and coefficients) to the ray's contribution list.

3.  **Pass 2: Sorting:**
    *   For each ray, sort all contributions in its list by knot position $t_k$ in increasing order [S1].

4.  **Pass 3: Accumulation and Compositing:**
    *   For each ray:
        *   Initialize current polynomial coefficients $a_{kd} = 0$ for all $d$ [G: Initial state before first knot].
        *   Iterate through sorted knots $(t_k, \hat{a}_{k0}, \dots, \hat{a}_{kD})$:
            *   Update coefficients for the current piece $k$ using the update rule [S1]:
                $$a_{kd} = \hat{a}_{kd} + \sum_{j=d}^D \binom{j}{d} a_{(k-1)j} (t_k - t_{k-1})^{j-d} \quad \text{for } d = 0, \dots, D$$
                where $t_{-1}$ is the ray start or previous knot, and $a_{(k-1)j}$ are coefficients from the previous piece.
            *   Evaluate the polynomial $A_k(t) = \sum_{d=0}^D a_{kd} (t - t_k)^d$ at sample points within $[t_k, t_{k+1}]$ (or integrate if using quadrature) to get the field value.
            *   Accumulate the field value along the ray for compositing (e.g., using standard DVR compositing equations with the computed field value as the density/opacity source).

### Parameters
*   $D$: Maximum polynomial degree (order of approximation).
*   $K$: Number of non-trivial polynomial pieces per particle.

### Edge Cases and Pitfalls
*   **Continuity and Symmetry:** Approximation candidates must be continuous, even functions with compact support to match the target $B_\Lambda$ [S1].
*   **Numerical Stability:** For $\Lambda$ close to the upper bound of the kernel support, evaluation of symbolic formulae becomes unstable. Use multiple precision arithmetic (e.g., GNU MPFR) for precomputation [S1].
*   **Degenerate Input:** If $\zeta = 0$, the particle contribution is undefined. Guard against zero smoothing radius.

## Operating characteristics
*   **Time Complexity:** The summation of piecewise polynomials reduces to a sorting task. The update rule for coefficients has constant complexity per piece, irrespective of the number of particles contributing [S1]. Precomputation is done offline.
*   **Memory Complexity:** Requires storage for the LUT (size depends on $\Lambda$ resolution, $D$, $K$) and per-ray contribution lists.
*   **Strengths:** Higher-order approximation provides better accuracy than linear/constant methods, adapting to data resolution. Efficient summation via sorting.
*   **Limitations:** Precomputation is complex and requires computer algebra systems. Numerical instability for large $\Lambda$.
*   **When to use:** When high-fidelity rendering of SPH data is required, and precomputation time is acceptable.
*   **When not to use:** For real-time rendering with strict latency constraints where precomputation is not feasible, or for very low-resolution approximations where linear methods suffice.

## Background
The method is based on the paper "Particle-Wise Higher-Order SPH Field Approximation for DVR" [S1]. SPH was introduced by Gingold and Monaghan (1977) and Lucy (1977) [S1]. Direct Volume Rendering (DVR) has a long-standing tradition (Drebin et al., 1988) [S1]. The cubic B-spline kernel is referenced from Rosswog (2009) [S1].

## Sources
[S1] Particle-Wise Higher-Order SPH Field Approximation for DVR, https://arxiv.org/abs/2401.02896v1, fetched 2026-10-26
