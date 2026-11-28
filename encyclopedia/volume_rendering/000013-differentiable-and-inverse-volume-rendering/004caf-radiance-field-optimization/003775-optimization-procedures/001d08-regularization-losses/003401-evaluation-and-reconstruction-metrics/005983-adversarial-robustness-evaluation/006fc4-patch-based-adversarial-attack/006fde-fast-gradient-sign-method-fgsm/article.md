# Fast Gradient Sign Method (FGSM)
Also called: Goodfellow et al. adversarial attack

## Overview
The Fast Gradient Sign Method (FGSM) is an adversarial attack that calculates the sign of the gradient of the cost function with respect to the input, scales it by a constant $\epsilon$, and adds it to the input [S1]. It allows for rapid generation of adversarial examples [S1].

## How it works
**Inputs and Outputs:**
- Input: original input $x$, target label $y$, loss function $J(x, y)$, and perturbation magnitude $\epsilon$ [S1].
- Output: adversarial input $x_{adv}$ [S1].

**Data Structures:** Not given in the sources.

**Steps:**
1. Compute the loss $J(x, y)$ for the original input $x$ and label $y$ [S1].
2. Compute the gradient of the loss with respect to the input, $\nabla_x J(x, y)$ [S1].
3. Compute the sign of the gradient, $\text{sign}(\nabla_x J(x, y))$ [S1].
4. Scale the sign of the gradient by the constant $\epsilon$ [S1].
5. Add the scaled perturbation to the original input to obtain $x_{adv}$ [S1].

**Formula:**
$$x_{adv} = x + \epsilon \cdot \text{sign}(\nabla_x J(x, y))$$

**Parameters:**
- $\epsilon$ (epsilon): A constant controlling the intensity of the noise [S1].

**Edge Cases and Pitfalls:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sign(x):
    return 1.0 if x > 0 else (-1.0 if x < 0 else 0.0)

def fgsm_attack(x, y, loss_fn, epsilon):
    # Compute loss for original input and label
    loss = loss_fn(x, y)
    # Compute gradient of loss w.r.t input (standard backpropagation)
    grad = compute_gradient(loss, x)
    # Compute sign of gradient element-wise
    sign_grad = [sign(g) for g in grad]
    # Scale by epsilon and add to input
    x_adv = [x[i] + epsilon * sign_grad[i] for i in range(len(x))]
    return x_adv
```

## Subtopics

### Fast Gradient Value Method
An extension of FGSM that uses a scaled version of the gradient's magnitude (value) rather than just the sign [S1]. Proposed by Rozsa et al. [S1].

### Iterative FGSM with Momentum
An iterative version of FGSM that incorporates momentum into the gradient update equation to avoid poor local minima and non-convex patterns in the objective function's landscape [S1]. Proposed by Dong et al. [S1].

## Operating characteristics

## Background
Proposed by Goodfellow et al. [S1], FGSM was one of the initial adversarial attacks designed to exploit the high-dimensional nature of neural network inputs [S1]. The vulnerability stems from this high dimensionality, which enables the generation of non-realistic input samples that produce outputs deviating drastically from human judgment and expected outcomes [S1]. The method allows for rapid generation of attacks [S1].

## Sources
[S1] Targeted Adversarial Attacks on Generalizable Neural Radiance Fields, https://doi.org/10.1109/iccvw60793.2023.00399, fetched 2026-10-29
