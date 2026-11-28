# Heaviside Step Derivative Approximation

Also called: Finite Difference Heaviside Derivative, Heaviside Derivative Approximation

## Overview
This method provides a gradient approximation for the Heaviside step function $H(g)$ within reverse-mode automatic differentiation, avoiding the undefined derivative at discontinuities by using a finite difference approximation derived from intermediate forward-pass values [S1]. It sets itself apart from standard Analytic Differentiation (AD) by handling discontinuous boolean logic and step functions where AD rules fail or require complex custom adjoints, while maintaining $O(1)$ complexity relative to the number of parameters, unlike naive finite difference which scales as $O(n)$ [S1].

## How it works
The method approximates the derivative of $H(g)$ by leveraging two intermediate scalar values, $g_+$ and $g_-$, computed during the forward pass. These values represent the argument $g$ evaluated at slightly perturbed positions or states, effectively creating a local finite difference stencil without requiring additional backward passes or explicit perturbation logic in the backward graph.

**Inputs and Outputs:**
- Input: The forward-computed intermediate values $g_+$ and $g_-$, and the forward output $H(g)$ (typically $H(g_+)$ or $H(g_-)$ depending on implementation, but the gradient uses the pair).
- Output: The gradient $\frac{\partial L}{\partial g}$ with respect to the scalar argument $g$, where $L$ is the loss.

**Data Structures:**
- $g_+$: Scalar float, the value of $g$ at the "positive" or perturbed sample.
- $g_-$: Scalar float, the value of $g$ at the "negative" or unperturbed sample.
- $\Delta g$: The difference $g_+ - g_-$. In the context of the source, this is implicitly handled by the finite difference quotient.

**Steps:**
1. **Forward Pass:** Compute $g_+$ and $g_-$ as intermediate values during the evaluation of $H(g)$. Specifically, the computation of $H(g_+)$ and $H(g_-)$ yields these intermediates [S1].
2. **Finite Difference Approximation:** Approximate the derivative $\frac{dH}{dg}$ using the difference in the Heaviside outputs divided by the difference in inputs. Since $H$ is a step function, $H(g_+) - H(g_-)$ is non-zero only if $g_+$ and $g_-$ straddle the discontinuity at 0.
3. **Gradient Propagation:** Multiply the incoming gradient $\frac{\partial L}{\partial H}$ by the approximated derivative.

**Formula:**
The derivative is approximated as:
$$ \frac{dH}{dg} \approx \frac{H(g_+) - H(g_-)}{g_+ - g_-} $$
Where:
- $H(x)$ is the Heaviside step function, defined as $1$ if $x \ge 0$ and $0$ otherwise (or similar convention).
- $g_+$ and $g_-$ are the intermediate values computed in the forward pass [S1].
- The denominator $g_+ - g_-$ acts as the step size for the finite difference.

**Parameters:**
- The separation between $g_+$ and $g_-$ is not explicitly defined as a tunable parameter $\epsilon$ in the source text, but is determined by the forward pass computation structure [S1]. The source notes that $g_+$ and $g_-$ are computed as intermediate values, implying their relationship is fixed by the program structure or a small perturbation applied during the forward evaluation [S1].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $g_+ = g_-$, the denominator is zero. This occurs if the perturbation is zero or if the forward pass does not actually produce distinct values. The source implies $g_+$ and $g_-$ are distinct intermediates [S1].
- **Multiple Discontinuities:** The method works correctly when there is at most one discontinuity in the local region. If intermediate values depend on the same discontinuity and interact (e.g., multiplication of two discontinuous vectors), the approximation may fail or require careful handling [S1].
- **Precision:** The approximation relies on the difference $H(g_+) - H(g_-)$ being non-zero, which requires $g_+$ and $g_-$ to straddle the threshold. Numerical precision issues may cause both to be on the same side of zero, resulting in a zero gradient even if a discontinuity was crossed.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def heaviside_step_derivative_backward(g_plus, g_minus, grad_out):
    """
    Computes the gradient for the Heaviside step function using finite difference.
    
    Args:
        g_plus (float): Intermediate value g+ computed in forward pass.
        g_minus (float): Intermediate value g- computed in forward pass.
        grad_out (float): Gradient of the loss with respect to H(g).
        
    Returns:
        float: Gradient of the loss with respect to g.
    """
    # Compute the difference in Heaviside outputs
    # H(x) is 1 if x >= 0, else 0
    h_plus = 1.0 if g_plus >= 0.0 else 0.0
    h_minus = 1.0 if g_minus >= 0.0 else 0.0
    
    # Compute the difference in inputs
    delta_g = g_plus - g_minus
    
    # Avoid division by zero
    if delta_g == 0.0:
        return 0.0
        
    # Approximate derivative dH/dg
    dH_dg = (h_plus - h_minus) / delta_g
    
    # Chain rule: dL/dg = dL/dH * dH/dg
    grad_in = grad_out * dH_dg
    
    return grad_in
```

## Subtopics

### [Function Composition Gradient Rule for Discontinuous Functions](20428-function-composition-gradient-rule-for-discontinuous-functio.md)
This method provides a gradient approximation for composite functions $f = h(g)$ where $h$ is continuous but $g$ is discontinuous, specifically addressing the failure of standard automatic differentiation (AD) at discontinuity boundaries. It sets itself apart from the parent [Heaviside Step Derivative Approximation](20427-heaviside-step-derivative-approximation.md) by generalizing the two-sided sampling principle to arbitrary continuous outer functions $h$, rather than limiting the approximation to the Heaviside step function itself. It differs from the [Surrogate Gradient Method](21618-surrogate-gradient-method.md) by deriving the gradient from a finite difference quotient of the composite function's branches rather than replacing the derivative with a heuristic surrogate.

### [Surrogate Gradient Method](21618-surrogate-gradient-method.md)
The Surrogate Gradient Method is a technique for optimizing systems containing non-differentiable operations, such as the Heaviside step function used in spiking neural networks, by replacing the undefined derivative with a differentiable approximation during back-propagation. It differs from exact adjoint methods by introducing a bias in the gradient estimate, allowing end-to-end training where standard automatic differentiation would fail due to discontinuities.

## Operating characteristics
- **Time Complexity:** $O(1)$ with respect to the number of parameters $n$ when combined with reverse-mode differentiation, similar to Analytic Differentiation (AD) [S1].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Slower than Analytic Differentiation (AD) due to more complex rules, but faster than finite difference which has $O(n)$ complexity [S1].
- **Strengths:** Handles discontinuous boolean logic and step functions without extra backward passes; efficient for programs with many parameters compared to finite difference.
- **Limitations:** Fails or requires special handling when multiple discontinuities interact (e.g., multiplication of discontinuous vectors); relies on the existence of distinct intermediate values $g_+$ and $g_-$ [S1].
- **When to use:** When differentiating through Heaviside step functions or boolean logic in reverse-mode AD, especially when AD rules are insufficient or complex.
- **When not to use:** When the local region contains multiple interacting discontinuities, or when higher precision analytic gradients are available and simpler.

## Background
The method is described in the paper "A δ" [S1]. It is motivated by the need to handle discontinuities in differentiable rendering and programming, where standard AD fails. The approach uses a 1D box kernel concept but implements it via finite difference on intermediate values to avoid extra back-propagation passes [S1].

## Sources
[S1] A δ, https://doi.org/10.1145/3528223.3530125, fetched 2026-10-30
