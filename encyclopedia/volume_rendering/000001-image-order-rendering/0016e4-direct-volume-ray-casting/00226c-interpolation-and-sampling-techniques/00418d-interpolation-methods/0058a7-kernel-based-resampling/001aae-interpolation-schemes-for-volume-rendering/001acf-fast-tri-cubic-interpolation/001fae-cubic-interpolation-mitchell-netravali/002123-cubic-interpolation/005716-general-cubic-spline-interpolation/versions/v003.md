# General Cubic Spline Interpolation
Also called: B-C Spline Family, General cubic-spline family, Mitchell-Netravali Spline Family

## Overview
General cubic spline interpolation defines a family of piecewise third-order polynomial reconstruction filters parameterized by $B$ and $C$ [S1, unofficial]. It solves the problem of smooth volume data sampling by providing a continuous function with a continuous first derivative for all parameter values, ensuring smoother results than linear interpolation [S1, unofficial]. It is distinct from its siblings, such as the four-parameter polynomial family, by utilizing third-order polynomials rather than fourth-order, which limits its ability to guarantee second-derivative continuity but simplifies the parameter space to two variables [S1, unofficial].

## Shared principles
The family is defined by a symmetric, piecewise cubic reconstruction filter $h(x)$ with support $[-2, 2]$ [S1, unofficial]. The function satisfies $h(x) = h(-x)$ for $x < 0$ [S1, unofficial]. For the positive half of the domain, the filter is composed of two cubic polynomials defined by free parameters $B$ and $C$ [S1, unofficial]:

$$
h(x) = 
\begin{cases} 
\left(\frac{2}{3} - \frac{B}{2} - C\right)x^3 + (-3 + 2B + C)x^2 + 1 - \frac{B}{2} & 0 \le x < 1 \\
\left(\frac{B}{2} - C\right)x^3 + (B + 5C)x^2 + \frac{4B}{3}x + \left(-2B - 8C\right)x + \frac{4B}{3} + 4C & 1 \le x < 2 \\
0 & \text{otherwise}
\end{cases}
$$

The function is continuous and has a continuous first derivative for all values of $B$ and $C$ [S1, unofficial]. It reconstructs constant values correctly regardless of the parameter values [S1, unofficial]. However, it does not guarantee interpolation (passing through sample points) or linear reconstruction (reconstructing straight lines correctly) unless specific constraints on $B$ and $C$ are met [S1, unofficial].

To achieve linear reconstruction (condition 6), the parameters must satisfy:
$$
B = -2C + 1
$$
Under this constraint, the derivative at $x=1$ is fixed to $-1/2$ [S1, unofficial]. To achieve interpolation (condition 4), the parameter $B$ must be zero [S1, unofficial].

## Subtopics

### [Four-parameter polynomial family](15871-four-parameter-polynomial-family.md)
A set of fourth-order polynomial reconstruction filters defined by a single parameter $a$. It generalizes the Catmull-Rom spline and allows for second-derivative continuity under specific parameter choices, a property not achievable with the third-order general cubic splines. Choose this when higher-order continuity is required.

## Background
The general cubic spline family was described by Mitchell and Netravali in 1988 [S1, unofficial]. It is commonly used in image processing and volume rendering to balance smoothness and sharpness through the tuning of parameters $B$ and $C$ [S1, unofficial]. The cubic B-spline is a notable instance within this family, corresponding to parameters $B=1$ and $C=0$, which results in an approximating rather than interpolating filter [S1, unofficial].

## Sources
[S1] Iso-Surface Volume Rendering : speed and accuracy for medical applications, https://doi.org/10.3990/1.9789036513982, fetched 2026-10-26 (unofficial)
