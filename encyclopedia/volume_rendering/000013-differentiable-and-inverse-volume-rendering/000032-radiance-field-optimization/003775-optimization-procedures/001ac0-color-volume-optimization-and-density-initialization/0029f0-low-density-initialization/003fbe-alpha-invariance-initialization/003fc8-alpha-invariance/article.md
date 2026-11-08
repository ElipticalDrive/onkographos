# Alpha Invariance

Also called: Inverse Scaling Analysis

## Overview
Alpha invariance is the property that volume rendering results (specifically the alpha compositing weights) remain invariant under arbitrary scene scaling, requiring volume density $\sigma$ to scale inversely with the ray segment length $d$ [S1, unofficial]. It solves the problem of optimization failure in neural radiance fields when scene scales change, where standard parameterizations either fail to produce sufficient density for small intervals or cause cloudiness for large intervals [S1, unofficial]. It sets itself apart from sibling methods by providing a theoretical foundation for density initialization and activation function selection, rather than just a heuristic initialization step.

## Shared principles
The core principle is that for a radiance field to render identical RGB colors regardless of the scene's physical scale $L$, the volume density $\sigma$ must satisfy an inverse scaling relationship with the distance $d$ [S1, unofficial].

### Rendering Equation and Alpha Compositing
The rendered color $\mathbf{c}$ is the expectation of the sample mean, approximated by discrete alpha compositing [S1, unofficial]:
$$
\mathbb{E}[\mathbf{c}(t)] = \int_0^\infty \sigma(t) T(t) \mathbf{c}(t) dt
$$
where $T(t) = \exp(-\int_0^t \sigma(s) ds)$ is the transmittance [S1, unofficial]. For a segment $i$ with length $d_i$ and constant density $\sigma_i$, the alpha value is:
$$
\alpha_i = 1 - e^{-\sigma_i d_i}
$$
The weight $w_i$ contributed by this segment is:
$$
w_i = \prod_{j < i} (1 - \alpha_j) \cdot \alpha_i
$$
To maintain alpha invariance, if the scene size expands by a factor $k$ (so $d' = k d$), the density must shrink by $1/k$ (so $\sigma' = \sigma / k$) to keep $\alpha$ constant [S1, unofficial].

### Activation Function Behavior
The mapping from a raw network output $x$ to density $\sigma$ is determined by the activation function. The resulting alpha $\alpha(x, d)$ behaves differently across common activations [S1, unofficial]:

1.  **ReLU**: $\alpha = 1 - \exp(-\text{ReLU}(x) \cdot d)$.
2.  **Softplus**: $\alpha = 1 - \text{sigmoid}(-x)^d$, derived from $\exp(-\text{softplus}(x)) = \text{sigmoid}(-x)$.
3.  **Exp**: $\alpha = 1 - \exp(-\exp(x) \cdot d)$.

The **Exp** activation is numerically stable and robust to scaling because the distance $d$ can be incorporated into the exponent as $\log d$:
$$
\alpha = 1 - \exp(-\exp(x + \log d)) = 1 - \text{GumbelCDF}(-(x + \log d))
$$
This formulation prevents overflow issues associated with large $\sigma$ in small intervals, a common pitfall for ReLU and Softplus [S1, unofficial].

### Empirical Inverse Scaling
Empirical analysis shows that the ratio of volume density $\sigma$ at different scaling factors $k$ is close to $1/k$ [S1, unofficial]. This is observed on object surfaces, defined as the 50th percentile location of the volume rendering CDF [S1, unofficial].

## Subtopics

### Alpha Invariance Visualization
A visualization technique that produces a "division image" to show the global difference in numerical range of $\sigma$ across different scene scaling factors $k$ [S1, unofficial]. It displays the $\sigma$-image at the 50-th percentile location of each ray's density histogram CDF [S1, unofficial]. Visualizations for TensoRF (ficus, ship, fern, horns) and Nerfacto (ballroom, museum) demonstrate that alpha invariance holds strongly across architectures and scenes [S1, unofficial].

## Background
The concept originates from the analysis of neural radiance fields (NeRF) and their sensitivity to scene scale [S1, unofficial]. It was identified as a critical factor in preventing optimization failure due to floaters or cloudiness when scenes are scaled [S1, unofficial]. The term "Alpha Invariance" was coined to describe the desired property of $\alpha$ invariance with respect to scene scaling [S1, unofficial]. The related term "Inverse Scaling Analysis" refers to the empirical method of analyzing how $\sigma$ changes with $k$ [S1, unofficial].

## Sources
[S1] Alpha Invariance: On Inverse Scaling Between Distance and Volume Density in Neural Radiance Fields, https://www.youtube.com/watch?v=YQhBEXC464w, fetched 2026-10-31 (unofficial)
