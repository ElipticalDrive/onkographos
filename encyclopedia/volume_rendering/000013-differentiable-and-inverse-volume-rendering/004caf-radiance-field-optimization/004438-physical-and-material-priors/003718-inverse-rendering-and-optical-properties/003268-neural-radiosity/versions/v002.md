# Neural Radiosity


## Overview
Neural Radiosity (NR) is an inverse rendering method that solves the rendering equation by representing the radiance field as a neural network and optimizing its parameters to minimize the residual of the equation [S1, unofficial]. Unlike standard differentiable rendering which backpropagates through a Monte Carlo graph, NR formulates the problem as a self-training optimization of the radiance function's residual, enabling gradient-based inverse rendering without storing the full computation graph of the renderer [S1, unofficial]. It differs from its siblings by directly approximating the solution to the integral equation rather than differentiating a discrete rasterization or path-tracing kernel [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry, material properties (BRDF $f$, emission $E$), and scene parameters $\mathbf{p}$ to be optimized.
*   **Output:** A neural network $L_\phi(\mathbf{x}, \boldsymbol{\omega}_o)$ representing outgoing radiance, and gradients $\nabla_\theta \mathcal{L}$ for optimization.

**Data Structures:**
*   **Radiance Network ($L_\phi$):** A Multi-Layer Perceptron (MLP) with 6 fully-connected layers and 512 neurons per hidden layer [S1, unofficial].
*   **Input Vector:** Concatenation of surface location $\mathbf{x}$, outgoing direction $\boldsymbol{\omega}_o$, surface normal, diffuse reflectance, and specular reflectance [S1, unofficial].
*   **Feature Grids:** Multi-resolution learnable feature vectors stored in sparse grids, concatenated with the input vector to capture high-frequency details [S1, unofficial].

**Formulas:**
The rendering equation is expressed as a residual $r_\phi(\mathbf{x}, \boldsymbol{\omega}_o)$ [S1, unofficial]:
$$
r_\phi(\mathbf{x}, \boldsymbol{\omega}_o) = L_\phi(\mathbf{x}, \boldsymbol{\omega}_o) - E(\mathbf{x}, \boldsymbol{\omega}_o) - \int_{H^2} f(\mathbf{x}, \boldsymbol{\omega}_i, \boldsymbol{\omega}_o) L_\phi(\mathbf{x}'(\mathbf{x}, \boldsymbol{\omega}_i), -\boldsymbol{\omega}_i) d\boldsymbol{\omega}_i
$$
The loss function $\mathcal{L}(\theta)$ is the squared $L_2$ norm of this residual integrated over the scene surface $M$ and hemisphere $H^2$ [S1, unofficial]:
$$
\mathcal{L}(\theta) = \| r_\theta(\mathbf{x}, \boldsymbol{\omega}_o) \|_2^2 = \int_M \int_{H^2} r_\theta(\mathbf{x}, \boldsymbol{\omega}_o)^2 d\mathbf{x} d\boldsymbol{\omega}_o
$$
This is estimated via Monte Carlo sampling with $N$ samples from distribution $p(\mathbf{x}, \boldsymbol{\omega})$ [S1, unofficial]:
$$
\mathcal{L}(\theta) \approx \frac{1}{N} \sum_{j=1}^N \frac{r_\theta(\mathbf{x}_j, \boldsymbol{\omega}_{o,j})^2}{p(\mathbf{x}_j, \boldsymbol{\omega}_{o,j})}
$$
The gradient of the loss with respect to network parameters $\theta$ is [S1, unofficial]:
$$
\nabla_\theta \mathcal{L}(\theta) \approx \frac{1}{N} \sum_{j=1}^N \frac{2 r_\theta(\mathbf{x}_j, \boldsymbol{\omega}_{o,j}) \nabla_\theta r_\theta(\mathbf{x}_j, \boldsymbol{\omega}_{o,j})}{p(\mathbf{x}_j, \boldsymbol{\omega}_{o,j})}
$$
The gradient of the residual $\nabla_\theta r_\theta$ involves the derivative of the radiance network and the BRDF/emission with respect to scene parameters $\mathbf{p}$ [S1, unofficial]:
$$
\nabla_\theta r_\theta(\mathbf{x}_j, \boldsymbol{\omega}_{o,j}) = \nabla_\theta (\partial_\mathbf{p} L_\theta(\mathbf{x}_j, \boldsymbol{\omega}_{o,j})) - \frac{1}{M} \sum_{k=1}^M \frac{f(\mathbf{x}_j, \boldsymbol{\omega}_{i,j,k}, \boldsymbol{\omega}_{o,j}) \nabla_\theta (\partial_\mathbf{p} L_\theta(\mathbf{x}'(\mathbf{x}_j, \boldsymbol{\omega}_{i,j,k}), -\boldsymbol{\omega}_{i,j,k}))}{p(\boldsymbol{\omega}_{i,j,k})}
$$
Note: The source text [S1, unofficial] presents a more complex residual gradient involving $\partial_\mathbf{p} f$ and $\partial_\mathbf{p} E$ terms in Equation 9, but Equation 11 simplifies the gradient of the residual for the loss update. The implementation uses automatic differentiation to compute $\partial_\mathbf{p} L_\theta$, $\partial_\mathbf{p} f$, and $\partial_\mathbf{p} E$.

**Steps:**
1.  **Initialize** the radiance network $L_\phi$ with random weights.
2.  **Sample** surface locations $\mathbf{x}_j$ uniformly from all scene surfaces $M$ and outgoing directions $\boldsymbol{\omega}_{o,j}$ uniformly from the unit hemisphere $H^2$ (or sphere for two-sided BSDFs) [S1, unofficial].
3.  **Compute Residual:** For each sample, evaluate $L_\phi(\mathbf{x}_j, \boldsymbol{\omega}_{o,j})$.
4.  **Estimate Incident Integral:** Sample $M$ incident directions $\boldsymbol{\omega}_{i,j,k}$ using importance sampling for emitter directions and BSDF distributions. Detach these samples from the computation graph to avoid correlation bias [S1, unofficial]. Query $L_\phi$ for incident radiance $L_\phi(\mathbf{x}'(\mathbf{x}_j, \boldsymbol{\omega}_{i,j,k}), -\boldsymbol{\omega}_{i,j,k})$.
5.  **Calculate Loss:** Compute the Monte Carlo estimate of $\mathcal{L}(\theta)$ using the residual [S1, unofficial].
6.  **Backpropagate:** Use automatic differentiation to compute $\nabla_\theta \mathcal{L}(\theta)$. This requires computing gradients of $L_\phi$ with respect to $\theta$ and scene parameters $\mathbf{p}$ [S1, unofficial].
7.  **Update:** Optimize network parameters $\theta$ (and potentially scene parameters $\mathbf{p}$) using stochastic gradient descent [S1, unofficial].

**Parameters:**
*   **Network:** MLP, 6 layers, 512 hidden units [S1, unofficial].
*   **Samples:** $N$ for surface/direction, $M$ for incident directions. Typical values not specified, but DNR uses 8 spp for LHS and 2048 for RHS [S1, unofficial].
*   **Sampling Distribution:** Uniform for $\mathbf{x}$ and $\boldsymbol{\omega}_o$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Correlation Bias:** Using the same Monte Carlo samples for both factors in the gradient estimation leads to $E(ab) \neq E(a)E(b)$ [S1, unofficial]. Mitigated by detaching incident direction samples [S1, unofficial].
*   **Memory Intensity:** Storing the full computation graph is memory intensive [S1, unofficial]. NR mitigates this by optimizing the residual directly, though AD is still used for gradient computation [S1, unofficial].
*   **Bias:** Keeping $\partial_\mathbf{p} L_\theta$ and $L_\phi$ outdated introduces bias, but experiments show convergence improvements [S1, unofficial].

## Subtopics

### [Differentiable Neural Radiosity](7410-differentiable-neural-radiosity.md)
A variant that optimizes a network for the *derivative* of radiance rather than radiance itself, enabling gradient-based inverse rendering without backpropagating through the Monte Carlo rendering graph. It provides view-independent solutions with constant time and memory complexity relative to path length.

## Operating characteristics
*   **Time Complexity:** Not explicitly given, but relies on Monte Carlo estimation with $N$ samples and $M$ incident directions per sample.
*   **Memory Complexity:** Lower than standard AD as it avoids storing the full computation graph of the renderer, but still requires storing the network and intermediate gradients [S1, unofficial].
*   **Strengths:** View-independent solutions, constant time/memory relative to path length, avoids quadratic/linear scaling of path-building methods [S1, unofficial].
*   **Limitations:** Correlation bias if not handled, memory intensive for complex scenes if not optimized, bias from outdated gradients [S1, unofficial].
*   **When to use:** Inverse rendering tasks requiring gradient-based optimization of scene parameters without the memory overhead of full differentiable path tracing [S1, unofficial].
*   **When not to use:** Scenes with extremely high-frequency details not captured by the network architecture, or when real-time performance is critical and network inference is too slow.

## Background
Neural Radiosity was introduced by Hadadan et al. in 2021 [S1, unofficial]. It is implemented in Mitsuba2 with Python bindings, utilizing its differentiable megakernel for automatic differentiation [S1, unofficial]. The method builds on the principle of minimizing the residual of the rendering equation using neural networks [S1, unofficial].

## Sources
[S1] Differentiable Neural Radiosity, https://arxiv.org/abs/2201.13190, fetched 2026-10-29 (unofficial)
