# Linear Approximation of Activation Functions for Probabilistic Affine Forms

Also called: Analytical least-squares activation approximation

## Overview
This method approximates nonlinear activation functions with linear functions to enable the analytical propagation of probabilistic affine forms (PAFs) through Implicit Neural Representations (INRs) [S1]. It solves the least-squares optimization problem analytically to determine the slope and intercept of the linear approximation, allowing uncertainty to be propagated efficiently without the conservative bounds associated with range analysis [S1]. It is distinct from range analysis by maintaining efficiency and accuracy for periodic functions like Sine, where range analysis suffers from high approximation error [S1].

## How it works
**Inputs and Outputs**
*   **Input:** The mean $\mu$ and standard deviation $\sigma$ of the estimated Gaussian probability density function (PDF) for the input probabilistic affine form [S1]. Note: The source text refers to $\sigma$ as "variance" in Table V captions, but the formulas and context define it as the standard deviation parameter of the Gaussian PDF [S1].
*   **Output:** Linear approximation coefficients $\alpha$ (slope) and $\beta$ (intercept), and the mean squared error $\gamma^2$ of the approximation [S1].

**Data Structures**
No complex data structures are required. The method computes scalar coefficients based on the input $\mu$ and $\sigma$.

**Steps**
1.  Identify the activation function type (ReLU, Sine, or ELU).
2.  Compute the intermediate variables $a$, $b$, $c$, and $\beta$ (for ELU) using the Gaussian error function $\text{erf}$ and exponential functions, as defined in the formulas below.
3.  Compute the linear approximation slope $\alpha$ and intercept $\beta_{linear}$ (denoted as $\beta$ in the source formulas for ReLU/Sine, but distinct from the intermediate $\beta$ in ELU) using the derived equations.
4.  Compute the mean squared error $\gamma^2$ to quantify the approximation quality [S1].

**Formulas**
The linear approximation is defined as $f(x) \approx \alpha x + \beta_{linear}$. The coefficients are derived by minimizing the least-squares error over the Gaussian distribution defined by $\mu$ and $\sigma$ [S1].

**1. ReLU Approximation**
$$ a = \text{erf}\left(\frac{-\mu-\sigma}{\sqrt{2}\sigma}\right) $$
$$ b = \left[1 + \text{erf}\left(\frac{-\mu-\sigma}{2\sigma}\right)\right]e^{\mu+2\sigma} $$
$$ \alpha = \frac{1}{2}(1 - a + b) $$
$$ \beta_{linear} = \text{Not given in the sources.} $$
$$ \gamma^2 = \frac{1}{4}(1 - a^2)(\mu^2 + \sigma^2) + a\mu\beta_{linear} - \beta_{linear}^2 $$
*Note: The source provides $\gamma^2$ involving $\beta$, but does not explicitly define $\beta_{linear}$ for ReLU in the table snippet. However, standard least-squares linear regression implies $\beta_{linear} = E[f(X)] - \alpha E[X]$. The source formula for $\gamma^2$ suggests $\beta$ is the intercept. If $\beta$ is not provided, the implementation must derive it or the source is incomplete. Based on the Sine formula structure, $\beta$ is likely the intercept. For ReLU, the source table lists $\alpha$ and $\gamma^2$ but omits $\beta$ in the visible text block. [G] In standard linear approximation $y = \alpha x + \beta$, $\beta$ is the intercept. The source formula for $\gamma^2$ uses $\beta$. We assume $\beta$ is the intercept. If the source does not give it, we cannot invent it. However, looking at the Sine formula, $\beta$ is explicitly calculated. For ReLU, it is missing. [G] A common derivation for ReLU linear approximation under Gaussian input yields $\beta = \mu \Phi(\frac{-\mu}{\sigma}) + \sigma \phi(\frac{-\mu}{\sigma}) - \alpha \mu$. Since this is not in the source, we mark it as missing.*

**2. Sine Approximation**
$$ a = \text{erf}\left(\frac{-\mu}{\sqrt{2}\sigma}\right) $$
$$ \beta_{linear} = \sqrt{\frac{2}{\pi}}e^{\frac{-\mu}{2\sigma^2}} $$
$$ \alpha = a \cos \mu $$
$$ \gamma^2 = -a^2 (\sigma^2 \cos^2 \mu + \sin^2 \mu) + \frac{1}{2} [1 - a^4 \cos (2\mu)] $$
*Note: The source table for Sine lists $\beta = \sqrt{\frac{2}{\pi}}e^{\frac{-\mu}{2\sigma^2}}$ and $\alpha = a \cos \mu$. The $\gamma^2$ formula in the table for Sine is distinct from ReLU/ELU.*

**3. ELU Approximation**
$$ a = \text{erf}\left(\frac{-\mu}{\sqrt{2}\sigma}\right) $$
$$ c = \sqrt{\frac{1}{2\pi}}e^{\frac{-\mu^2}{2\sigma^2}} $$
$$ \beta_{intermediate} = \frac{1}{2}b(1 - \mu) - \frac{1}{2}(1 + a) + c\sigma $$
*Note: The source formula for ELU $\beta$ depends on $b$. The source text for ELU in Table V is fragmented. It lists $a$, $c$, and a formula for $\beta$ involving $b$. It does not define $b$ for ELU in the visible snippet. [G] In ELU, $f(x) = \alpha(x-1)$ for $x<0$ and $x$ for $x \ge 0$. The variable $b$ in the source likely refers to an integral term similar to ReLU. Without the definition of $b$ for ELU, the formula is incomplete. [G] Assuming $b$ is defined similarly to ReLU or is a typo for a specific integral, we cannot implement it without the definition. However, the source lists $\alpha = \frac{1}{2}(1 - a)$. This seems inconsistent with standard ELU. The source text is the authority. We must use the source's $\alpha$. The $\beta$ formula uses $b$. If $b$ is not defined for ELU, we cannot compute $\beta$. [G] The source table V snippet for ELU is: `a = erf(...)`, `c = ...`, `beta = 1/2 b(1-mu) - ...`. It does not define `b` for ELU. This is a gap in the source.*
$$ \alpha = \frac{1}{2}(1 - a) $$
$$ \gamma^2 = \frac{1}{4}(1 - a^2)(\mu^2 + \sigma^2) + \left(\frac{1}{2}ab - c^2 - \frac{1}{4}b^2 - \frac{1}{2}b\right)\sigma^2 + ac\mu\sigma + (a - b + 1)c\sigma + \frac{1}{2}(ab - b - a^2 + 1)\mu + \frac{1}{4}\left[1 + \text{erf}\left(\frac{-\mu-2\sigma}{2\sigma}\right)\right]e^{2\mu+2\sigma} + \frac{1}{4}(a - b) - \frac{1}{4}b $$

**Parameters**
*   $\mu$: Mean of the estimated Gaussian PDF for the input probabilistic affine form [S1].
*   $\sigma$: Standard deviation of the estimated Gaussian PDF for the input probabilistic affine form [S1].

**Edge Cases and Pitfalls**
*   **Division by Zero:** If $\sigma = 0$, the terms involving $\sigma$ in the denominator of the error function arguments become undefined. The method assumes a probabilistic distribution with non-zero variance [S1].
*   **Periodic Functions:** Range analysis is less efficient on periodic functions like Sine due to high approximation error and conservative bounds, whereas this linear approximation remains efficient [S1].
*   **Missing Definitions:** The source text for ELU and ReLU contains fragmented formulas (e.g., undefined $b$ in ELU, missing $\beta$ in ReLU). Implementers must refer to the full paper [S1] for complete derivations or use standard least-squares linear regression formulas for the specific activation function under Gaussian inputs if the source is incomplete.

## Operating characteristics
*   **Time Complexity:** Analytical solution, $O(1)$ per activation function evaluation.
*   **Memory Complexity:** $O(1)$ additional memory for coefficients.
*   **Performance:** Uncertainty propagation (UP) using this method is more efficient than range analysis (RA) under all tested activation functions [S1]. The benefit is most significant for Sine functions [S1].
    *   ReLU: UP 0.091s vs RA 0.120s [S1].
    *   Sine: UP 0.099s vs RA 0.235s [S1].
    *   ELU: UP 0.096s vs RA 0.121s [S1].
*   **Strengths:** Efficient for periodic functions (Sine) where range analysis fails; analytical solution avoids iterative optimization.
*   **Limitations:** Approximation error $\gamma^2$ exists; relies on the assumption that the input PAF can be approximated by a Gaussian PDF [S1].

## Background
The method originates from the paper "Improving Efficiency of Iso-Surface Extraction on Implicit Neural Representations Using Uncertainty Propagation" [S1]. It was developed to address the inefficiency of range analysis when propagating uncertainty through nonlinear activation functions in INRs, particularly for periodic activations like Sine [S1]. It builds on the concept of Propagation of Affine Forms (PAF) [18502-propagation-of-affine-forms-paf.md].

## Sources
[S1] Improving Efficiency of Iso-Surface Extraction on Implicit Neural Representations Using Uncertainty Propagation, https://doi.org/10.1109/tvcg.2024.3365089, fetched 2026-10-26
