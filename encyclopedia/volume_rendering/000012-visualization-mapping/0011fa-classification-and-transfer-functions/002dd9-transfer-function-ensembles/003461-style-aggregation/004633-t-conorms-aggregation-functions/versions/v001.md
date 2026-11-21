# T-conorms (Aggregation Functions)
Also called: Probabilistic Sum T-conorm

## Overview
T-conorms (triangular conorms) are binary aggregation functions used to combine face occupancy probabilities into a single aggregated value per pixel in differentiable volume rendering. They serve as the continuous, real-valued generalization of the logical 'or' operator, allowing for differentiable occlusion testing by ensuring the aggregation function is continuous and differentiable. Unlike the discontinuous maximum operator, T-conorms provide smooth gradients necessary for gradient-based optimization in rendering pipelines.

## Shared principles
A T-conorm is a binary operation $\perp: [0, 1] \times [0, 1] \rightarrow [0, 1]$ that satisfies four fundamental axioms [S1]:
1.  **Associativity**: $\perp(a, \perp(b, c)) = \perp(\perp(a, b), c)$
2.  **Commutativity**: $\perp(a, b) = \perp(b, a)$
3.  **Monotonicity**: $(a \le c) \land (b \le d) \Rightarrow \perp(a, b) \le \perp(c, d)$
4.  **Identity Element**: $0$ is the neutral element, such that $\perp(a, 0) = a$

T-conorms are the duals of T-norms (triangular norms). While the largest T-norm is the minimum function, the smallest T-conorm is the maximum function [S1]. For any T-conorm $\perp$, it holds that $\perp(a, b) \ge \max(a, b)$ [S1].

In the context of differentiable rendering, the T-conorm aggregates the occupancy probabilities of all faces $T$ for a given pixel. If $p_t$ represents the occupancy probability of face $t$ (derived from a sigmoid function of the signed distance to the face boundary), the aggregated occupancy $A_O(p, T)$ is computed as [S1]:

$$ A_O(p, T) = \bigoplus_{t \in T} p_t $$

where $\bigoplus$ denotes the iterative application of the chosen T-conorm. For the probabilistic sum, this simplifies to the probability that at least one face covers the pixel assuming stochastic independence: $1 - \prod_{t \in T} (1 - p_t)$ [S1].

To be suitable for differentiable rendering, the T-conorm must be continuously differentiable. The drastic T-conorm, defined as $\perp_\circ(a,b)=1$ for $(a,b)\in(0,1)^2$, is excluded because it yields zero gradients in the interior of the unit square, making it unsuitable for gradient-based optimization [S1].

## Subtopics

### Probabilistic Sum
The probabilistic sum is the standard aggregation function used in existing approximate differentiable renderers such as SoftRas and DIB-R [S1]. It corresponds to the probability that at least one out of two independent events occurs [S1].
*   **Formula**: $\perp_P(a, b) = a + b - ab$
*   **Properties**: Continuous, continuously differentiable, strict, Archimedean.
*   **When to choose**: It is the default choice for assuming stochastic independence of face coverages [S1].

### Einstein Sum
The Einstein sum models the relativistic addition of velocities under special relativity [S1].
*   **Formula**: $\perp_E(a, b) = \frac{a + b}{1 + ab}$
*   **Properties**: Continuous, continuously differentiable, strict, Archimedean.
*   **When to choose**: It serves as an alternative to the probabilistic sum, particularly when modeling physical constraints analogous to relativistic velocity addition [S1]. Note that $\perp_E(a, b)$ is equivalent to the Hamacher T-conorm with parameter $p=2$ [S1].

### Maximum
The maximum T-conorm is defined as $\perp_M(a, b) = \max(a, b)$ [S1].
*   **Formula**: $\perp_M(a, b) = \max(a, b)$
*   **Properties**: Continuous, but **not** continuously differentiable (discontinuous gradient at $a=b$). It is idempotent.
*   **When to choose**: It represents the logical 'or' equivalent. While it is the smallest T-conorm, its lack of continuous differentiability makes it less suitable for smooth gradient-based optimization compared to other T-conorms, though it is often included for comparison [S1].

### Hamacher T-conorm
A parameterized family of T-conorms that generalizes both the probabilistic sum ($p=1$) and the Einstein sum ($p=2$) [S1].
*   **Formula**: $\perp_H^p(a, b) = \frac{a+b+(p-2)ab}{1+(p-1)ab}$
*   **Parameter Range**: $p \in (0, \infty)$ [S1].
*   **Properties**: Continuous, continuously differentiable, strict, Archimedean.
*   **When to choose**: Useful for interpolating between probabilistic and Einstein behaviors by tuning $p$.

### Frank T-conorm
A parameterized family of T-conorms derived from the Frank T-norm.
*   **Formula**: $\perp_F^p(a, b) = 1 - \log_p \left( 1 + \frac{(p^{1-a}-1)(p^{1-b}-1)}{p-1} \right)$
*   **Parameter Range**: $p \in (0, \infty)$ [S1].
*   **Properties**: Continuous, continuously differentiable, strict, Archimedean.
*   **When to choose**: Provides a smooth transition between the probabilistic sum and the maximum T-conorm as $p$ varies.

### Yager T-conorm
A parameterized family of T-conorms.
*   **Formula**: $\perp_Y^p(a, b) = \min \left( 1, (a^p + b^p)^{1/p} \right)$
*   **Parameter Range**: $p \in (0, \infty)$ [S1].
*   **Properties**: Continuous, but **not** continuously differentiable everywhere (non-smooth, plateaus outside the unit circle for certain $p$). It is nilpotent.
*   **When to choose**: Use with caution in differentiable rendering due to non-smoothness; however, it is included in benchmarks for comparison [S1].

### Aczél-Alsina T-conorm
A parameterized family of T-conorms.
*   **Formula**: $\perp_A^p(a, b) = 1 - \exp \left( - \left( |\log(1-a)|^p + |\log(1-b)|^p \right)^{1/p} \right)$
*   **Parameter Range**: $p \in (0, \infty)$ [S1].
*   **Properties**: Continuous, continuously differentiable, strict, Archimedean.
*   **When to choose**: Offers alternative aggregation behavior with smooth gradients.

### Dombi T-conorm
A parameterized family of T-conorms.
*   **Formula**: $\perp_D^p(a, b) = 1 - \left( \left( \frac{1-a}{a} \right)^p + \left( \frac{1-b}{b} \right)^p \right)^{-1/p}$
*   **Parameter Range**: $p \in (0, \infty)$ [S1].
*   **Properties**: Continuous, continuously differentiable, strict, Archimedean.
*   **When to choose**: Provides flexible aggregation shapes controlled by $p$.

### Schweizer-Sklar T-conorm
A parameterized family of T-conorms.
*   **Formula**: $\perp_S^p(a, b) = 1 - ((1-a)^p + (1-b)^p - 1)^{1/p}$
*   **Parameter Range**: $p \in (-\infty, 0)$ [S1].
*   **Properties**: Continuous, continuously differentiable, strict, Archimedean.
*   **When to choose**: Useful for specific aggregation behaviors defined by negative parameters.

### Comparison Table
The following table summarizes the properties of selected T-conorms as benchmarked in GenDR [S1].

| T-conorm | Continuous | Continuously Differentiable | Strict | Idempotent | Nilpotent | Archimedean | Parameter Range |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Maximum ($\perp_M$) | Yes | No | No | Yes | No | No | - |
| Probabilistic ($\perp_P$) | Yes | Yes | Yes | No | No | Yes | - |
| Einstein ($\perp_E$) | Yes | Yes | Yes | No | No | Yes | - |
| Hamacher ($\perp_H^p$) | Yes | Yes | Yes | No | No | Yes | $p \in (0, \infty)$ |
| Frank ($\perp_F^p$) | Yes | Yes | Yes | No | No | Yes | $p \in (0, \infty)$ |
| Yager ($\perp_Y^p$) | Yes | No | No | No | Yes | Yes | $p \in (0, \infty)$ |
| Aczél-Alsina ($\perp_A^p$) | Yes | Yes | Yes | No | No | Yes | $p \in (0, \infty)$ |
| Dombi ($\perp_D^p$) | Yes | Yes | Yes | No | No | Yes | $p \in (0, \infty)$ |
| Schweizer-Sklar ($\perp_S^p$) | Yes | Yes | Yes | No | No | Yes | $p \in (-\infty, 0)$ |

Note: The drastic T-conorm is excluded due to zero gradients [S1]. The Yager T-conorm is non-smooth [S1].

## Background
T-conorms originate from the field of fuzzy logic and real-valued logics, where they serve as multi-valued generalizations of the logical disjunction ('or') [S1]. In the context of differentiable rendering, they were generalized and extensively benchmarked in the GenDR (Generalized Differentiable Renderer) paper [S1]. The GenDR paper demonstrated that replacing the standard probabilistic sum with other continuous T-conorms allows for a wide variety of differentiable rendering behaviors, resulting in over 1,200 concrete differentiable renderer instances when combined with various sigmoid functions [S1]. The research highlighted that while the probabilistic sum is the standard in renderers like SoftRas and DIB-R, other T-conorms like the Einstein sum offer viable alternatives with different gradient properties [S1].

## Sources
[S1] GenDR: A Generalized Differentiable Renderer, https://doi.org/10.1109/cvpr52688.2022.00397, fetched 2026-10-26
