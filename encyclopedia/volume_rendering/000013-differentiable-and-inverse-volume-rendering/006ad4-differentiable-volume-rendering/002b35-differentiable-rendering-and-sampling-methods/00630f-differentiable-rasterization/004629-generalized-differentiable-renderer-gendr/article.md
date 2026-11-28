# Generalized Differentiable Renderer (GenDR)

Also called: GenDR (Generalized Differentiable Renderer)

## Overview
GenDR is a generalized differentiable mesh renderer that models the forward rendering process under a probabilistic perturbation defined by a probability distribution with density $f$ and a temperature/scale parameter $\tau > 0$, making the renderer continuous everywhere and differentiable almost everywhere [S1]. It computes pixel occupancy/coverage by aggregating face contributions using a sigmoid function (CDF of a probability distribution) and a T-conorm for aggregation [S1]. Unlike hard renderers which are discrete and non-differentiable, GenDR provides meaningful gradients for gradient-based optimization tasks such as shape optimization [S1]. It generalizes existing differentiable renderers like SoftRas and DIB-R by allowing an array of different smoothing distributions [S1].

## How it works
**Inputs:**
*   A set of faces $T$ (typically triangles) representing the 3D mesh [S1].
*   A pixel coordinate $p$ in the image space [G].
*   A temperature/scale parameter $\tau > 0$ [S1].
*   A probability distribution with cumulative distribution function (CDF) $F$ [S1].
*   A T-conorm operator $\perp$ [S1].

**Outputs:**
*   Occupancy/coverage value $AO(p, T) \in [0, 1]$ for each pixel $p$ [S1].
*   Gradients $\frac{\partial AO}{\partial \text{mesh parameters}}$ for optimization [G].

**Data Structures:**
*   **Faces ($T$):** A list of triangles. Each triangle is defined by its vertices in 3D space [G].
*   **Distance ($d(p, t)$):** The distance from pixel $p$ to face $t$, projected into image space [S1].
*   **T-conorm ($\perp$):** A binary operation satisfying associativity, commutativity, monotonicity, and having 0 as a neutral element [S1].

**Numbered Steps:**

1.  **Transform to Image Space:** Transform all objects (mesh vertices) into image space via a sequence of affine transformations followed by camera projection [S1]. This step is inherently differentiable [S1].
2.  **Differentiable Occlusion Test:** For each pixel $p$, compute the set of faces covering it by evaluating a differentiable occlusion test for each face $t \in T$ [S1]. This is done by computing the distance $d(p, t)$ from the pixel to the face and applying the sigmoid function $F$ normalized by the temperature $\tau$ [S1].
    *   Compute the normalized distance: $x = d(p, t) / \tau$ [S1].
    *   Apply the CDF: $p_t = F(x) = F(d(p, t)/\tau)$ [S1].
3.  **Aggregate Occupancy:** Aggregate the occupancy probabilities $p_t$ of all faces for each pixel using a continuous T-conorm $\perp$ [S1]. The final occupancy $AO(p, T)$ is computed as:
    $$ AO(p, T) = \perp_{t \in T} F\left(\frac{d(p, t)}{\tau}\right) $$
    where $\perp$ is applied iteratively over all faces $t \in T$ [S1].
4.  **Shading (Optional):** Apply a differentiable shading model (e.g., Phong) to color the faces [S1]. If depth is required, use Gumbel depth perturbations (softmin) for continuity, or take the closest triangle [S1].

**Formulas:**
*   **Occupancy:**
    $$ AO(p, T) = \perp_{t \in T} F\left(\frac{d(p, t)}{\tau}\right) $$
    [S1]
*   **T-conorm Definition:** A binary operation $\perp : [0, 1] \times [0, 1] \to [0, 1]$ satisfying:
    *   Associativity: $\perp(a, \perp(b, c)) = \perp(\perp(a, b), c)$
    *   Commutativity: $\perp(a, b) = \perp(b, a)$
    *   Monotonicity: $(a \le c) \land (b \le d) \implies \perp(a, b) \le \perp(c, d)$
    *   Neutral Element: $\perp(a, 0) = a$
    [S1]
*   **Probabilistic Sum (Common T-conorm):** $\perp_P(a, b) = a + b - ab = 1 - (1-a)(1-b)$ [S1].

**Parameters:**
*   **$\tau$ (tau):** Temperature or scale parameter for the sigmoid function. Typical values in experiments: $\tau \in \{10^{-0.1 \cdot n} \mid n \in \mathbb{N}, 0 \le n \le 80\}$ [S1]. A fine-grained grid search over $\tau$ is critical as behavior differs for each distribution [S1].
*   **Distribution $F$:** The CDF of the chosen probability distribution (e.g., Logistic, Gaussian, Uniform, Cauchy, Laplace) [S1].
*   **T-conorm $\perp$:** The aggregation function (e.g., Probabilistic Sum, Maximum, Hamacher, Frank, Yager) [S1].

**Pitfalls and Edge Cases:**
*   **Dirac Delta:** The Dirac delta distribution corresponds to the Heaviside function, which yields a discrete, non-differentiable renderer and is unsuitable for gradient-based optimization [S1].
*   **Discontinuous T-conorms:** Discontinuous T-conorms (e.g., drastic T-conorm) are not suitable for differentiable renderers [S1].
*   **Finite Support:** Distributions with finite support (e.g., Uniform) only provide meaningful gradients in a limited proximity to each face [S1].
*   **Squared Distributions:** Regular distributions $F$ typically perform better than their squared counterparts $F^2$, except for Cauchy and reciprocal sigmoid which have linear convergence rates [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid_logistic(x):
    """Logistic sigmoid function (CDF of Logistic distribution)."""
    return 1.0 / (1.0 + math.exp(-x))

def t_conorm_probabilistic_sum(a, b):
    """Probabilistic sum T-conorm: a + b - a*b."""
    return a + b - a * b

def compute_face_pixel_distance(face, pixel_p):
    """[G] Computes the screen-space distance from pixel p to face t.
    This is a standard geometric rasterization step (e.g., barycentric coordinates or signed distance to the projected triangle plane)."""
    # Standard geometric computation omitted; returns float distance.
    return 0.0

def compute_occupancy(pixel_p, faces, tau, cdf_func, t_conorm_func):
    """
    Compute occupancy for a pixel using GenDR.
    
    Args:
        pixel_p: 2D coordinates of the pixel.
        faces: List of face objects, each having a method screen_distance(p).
        tau: Temperature/scale parameter.
        cdf_func: Cumulative distribution function F(x).
        t_conorm_func: T-conorm aggregation function.
        
    Returns:
        Occupancy value AO(p, T).
    """
    occupancy = 0.0
    
    for face in faces:
        dist = compute_face_pixel_distance(face, pixel_p)
        normalized_dist = dist / tau
        prob = cdf_func(normalized_dist)
        occupancy = t_conorm_func(occupancy, prob)
        
    return occupancy
```

## Subtopics

### Soft Rasterizer
A differentiable rasterizer that uses a probabilistic sum T-conorm and specific smoothing distributions, generalized by GenDR. [Soft Rasterizer](6779-soft-rasterizer.md)

### Ray Marching with Differential Diffusion
A differentiable volume rendering method that uses differential diffusion to compute gradients for ray marching. [Ray Marching with Differential Diffusion](3547-ray-marching-with-differential-diffusion.md)

### Differentiable Monte Carlo Ray Tracing
A differentiable path tracing method that uses Monte Carlo sampling to compute gradients for ray tracing. [Differentiable Monte Carlo Ray Tracing](8822-differentiable-monte-carlo-ray-tracing.md)

### RGBXY Derivatives with Optimal Transport (DROT)
A differentiable renderer that uses RGBXY parameterization and optimal transport to compute gradients for rendering. [RGBXY Derivatives with Optimal Transport (DROT)](8830-rgbxy-derivatives-with-optimal-transport-drot.md)

### Sensitivity AOV
A method for computing sensitivity analysis over arbitrary outputs in rendering pipelines. [Sensitivity AOV](10809-sensitivity-aov.md)

### SAR Differentiable Rendering Model
A differentiable rendering model based on SAR principles for gradient-based optimization. [SAR Differentiable Rendering Model](12650-sar-differentiable-rendering-model.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1]
*   **Memory Complexity:** Not given in the sources. [S1]
*   **Measured Performance:** Evaluated on ShapeNet 3D reconstruction benchmark; the uniform distribution yielded the best overall results when averaged across distributions [S1].
*   **Scaling:** Performance depends on the choice of distribution and T-conorm; regular distributions $F$ typically perform better than squared counterparts $F^2$, except for Cauchy and reciprocal sigmoid [S1].
*   **Strengths:** Provides continuous and differentiable rendering, enabling gradient-based optimization; generalizes existing methods like SoftRas and DIB-R [S1].
*   **Limitations:** Computationally expensive due to evaluating all faces for all pixels; choice of $\tau$ and distribution is critical and requires grid search [S1].
*   **When to Use:** Shape optimization tasks, e.g., optimizing a mesh to fit a silhouette within 100 steps [S1].
*   **When Not to Use:** Not given in the sources. [S1]

## Background
GenDR was introduced by Bastian Goldluecke and Felix Petersen from the University of Konstanz in 2022 [S1]. The work formalizes the requirements for differentiable rendering components and instantiates a general differentiable renderer with an array of different smoothing distributions [S1]. It generalizes existing differentiable renderers like SoftRas and DIB-R [S1]. The method is used in shape optimization tasks, such as optimizing a mesh to fit a silhouette [S1].

## Sources
[S1] GenDR: A Generalized Differentiable Renderer, https://doi.org/10.1109/cvpr52688.2022.00397, fetched 2026-10-26
