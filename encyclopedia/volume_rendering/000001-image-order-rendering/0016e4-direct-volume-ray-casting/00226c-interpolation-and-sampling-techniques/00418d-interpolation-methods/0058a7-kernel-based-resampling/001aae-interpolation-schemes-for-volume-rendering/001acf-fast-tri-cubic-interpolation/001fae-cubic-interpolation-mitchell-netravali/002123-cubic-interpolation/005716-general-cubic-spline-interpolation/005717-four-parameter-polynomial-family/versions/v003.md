# Four-parameter polynomial family

Also called: Four-parameter polynomial, Generalized Catmull-Rom family

## Overview
The four-parameter polynomial family is a set of fourth-order polynomial reconstruction filters defined by a single parameter $a$ [S1, unofficial]. It generalizes the Catmull-Rom spline, which corresponds to the specific case where $a=0$ [S1, unofficial]. This family is distinct from its parent, the general cubic spline, by utilizing fourth-order polynomials to allow for the continuity of the second derivative under specific parameter choices, a property not achievable with cubic splines [S1, unofficial].

## Shared principles
The family defines a symmetric, piecewise fourth-order reconstruction filter $h(x)$ with support $[-2, 2]$. The function is defined such that $h(x) = h(-x)$ for $x < 0$. For the positive half of the domain, the filter is composed of two fourth-order polynomials:

$$
h(x) = 
\begin{cases} 
ax^4 + \left(a - \frac{5}{2}\right)x^3 + \left(a - \frac{3}{2}\right)x^2 + 1 & 0 \le x < 1 \\
ax^4 + \left(6a - \frac{1}{2}\right)x^3 + \left(-13a + \frac{5}{2}\right)x^2 + (12a - 4)x + (2 - 4a) & 1 \le x < 2 \\
0 & \text{otherwise}
\end{cases}
$$

The parameter $a$ is a free variable that shapes the filter [S1, unofficial]. The function is continuous and has a continuous first derivative for all values of $a$ [S1, unofficial]. However, the second derivative is generally not continuous [S1, unofficial].

Specific constraints on $a$ yield notable properties:
1.  **Catmull-Rom Interpolation:** When $a=0$, the function reduces to the Catmull-Rom cubic-spline interpolation function [S1, unofficial].
2.  **Continuous Second Derivative:** When $a=-1/2$, the second derivative of the function becomes continuous [S1, unofficial]. This is the only polynomial in this specific fourth-order family that achieves second-derivative continuity while maintaining the ability to reconstruct a straight line [S1, unofficial].

## Subtopics

### Catmull-Rom Spline
The Catmull-Rom spline is the subset of the four-parameter polynomial family where $a=0$ [S1, unofficial]. It is a widely used interpolating cubic spline. It is chosen when smooth interpolation through sample points is required without the computational overhead of higher-order derivatives. See [Catmull-Rom](52-catmull-rom.md).

### B-Spline (Cubic)
While not a member of the four-parameter polynomial family (which is fourth-order), the cubic B-spline is mentioned in the context of this family as the only cubic-spline function that has a continuous second derivative and can reconstruct a straight line [S1, unofficial]. It serves as a comparison point for smoothness properties. See [B-Spline](52-b-spline.md).

## Background
The four-parameter polynomial family was introduced in the context of iso-surface volume rendering to balance speed and accuracy for medical applications [S1, unofficial]. The formulation allows for the derivation of reconstruction filters using more than four sample points, although this significantly increases the complexity of the 3D algorithm [S1, unofficial]. The family highlights the trade-off between derivative continuity and polynomial order, noting that while cubic splines are common, fourth-order polynomials offer additional degrees of freedom for smoothness constraints [S1, unofficial].

## Sources

[S1] Iso-Surface Volume Rendering : speed and accuracy for medical applications, https://doi.org/10.3990/1.9789036513982, fetched 2026-10-26 (unofficial)
