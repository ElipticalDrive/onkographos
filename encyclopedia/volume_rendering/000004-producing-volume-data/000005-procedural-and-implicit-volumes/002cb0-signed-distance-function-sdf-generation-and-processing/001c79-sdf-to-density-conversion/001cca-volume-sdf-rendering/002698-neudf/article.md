# NeUDF

Also called: Neural Unsigned Distance Fields, NeUDF*

## Overview
NeUDF is a neural rendering framework that reconstructs surfaces with arbitrary topologies, including open surfaces, from multi-view 2D image supervision by leveraging the Unsigned Distance Function (UDF) as the implicit surface representation [S1]. Unlike its sibling [Volume SDF Rendering](1344-volume-sdf-rendering.md), which is limited to closed (watertight) surfaces, NeUDF resolves the ambiguity of surface orientation in open structures by introducing a specialized "Udensity" probability density function and a normal regularization strategy [S1]. It differs from naive extensions of SDF-based rendering by avoiding spurious surfaces in void regions through an unbiased, occlusion-aware weighting paradigm tailored for UDFs [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of multi-view 2D images $\{I_k\}$ with known camera parameters (origin $o$, view direction $v$).
*   **Outputs:** A neural network $\psi$ that predicts the unsigned distance $d$ and color $c$ for any spatial location $x$ along a ray, and the reconstructed surface $\mathcal{S}_{\mathcal{O}}$ defined by the zero-level set of $d$.

**Data Structures:**
*   **Neural Network $\psi$:** A multi-layer perceptron (MLP) or similar differentiable network.
    *   **Input:** View direction $v \in \mathbb{S}^2$ and spatial location $x \in \mathbb{R}^3$.
    *   **Output:** Unsigned distance $d \in \mathbb{R}^+$ and color $c \in [0, 1]^3$.
    *   **Signature:** $(d, c) = \psi(v, x) : \mathbb{S}^2 \times \mathbb{R}^3 \mapsto (\mathbb{R}^+, [0, 1]^3)$ [S1].
*   **Udensity Function $\varsigma_r(\cdot)$:** A cumulative distribution function (CDF) parameterized by a learnable scalar $r$. It maps the UDF value $d$ to a probability mass.
    *   **Constraints:**
        1.  $\varsigma_r(0) = 0$ [S1].
        2.  $\lim_{d \rightarrow +\infty} \varsigma_r(d) = 1$ [S1].
        3.  First derivative $c'_r(d) > 0$ for $d > 0$ [S1].
        4.  Second derivative $c''_r(d) < 0$ for $d > 0$ (concave) [S1].
    *   **Note:** The specific functional form of $\varsigma_r$ is not explicitly defined in the provided text, but it must satisfy the above boundary and convexity conditions [S1]. The parameter $r$ controls the density distribution shape [S1].

**Numbered Steps:**

1.  **Ray Sampling:** For each pixel, cast a ray from camera origin $o$ with direction $v$. Sample points $p(t)$ along the ray at parameter $t \in [0, +\infty)$.
2.  **Network Query:** For each sample point $p(t)$, query the neural network $\psi$ using the ray direction $v$ and position $p(t)$ to obtain the unsigned distance $d = \Psi(p(t))$ and color $c$ [S1].
    *   The predicted surface is defined as $\mathcal{S}_{\mathcal{O}} = \{x \in \mathbb{R}^3 | d = 0, (d, c) = \psi(v, x)\}$ [S1].
3.  **Udensity Calculation:** Compute the opaque density $\tau_r(t)$ using the Udensity function $\varsigma_r$ and the chain rule for the derivative along the ray [S1]:
    $$ \tau_r(t) = \left| \frac{\frac{\partial(\varsigma_r \circ \Psi \circ p)}{\partial t}(t)}{\varsigma_r \circ \Psi \circ p(t)} \right| $$
    Here, $\circ$ denotes function composition. The term $\varsigma_r \circ \Psi \circ p(t)$ represents $\varsigma_r(d(t))$ [S1].
4.  **Weight Calculation:** Compute the rendering weight $w_r(t)$ using the standard volume rendering accumulation formula with the opaque density $\tau_r(t)$ [S1]:
    $$ w_r(t) = \tau_r(t) e^{-\int_0^t \tau_r(u) du} $$
    This weight function is unbiased and occlusion-aware [S1].
5.  **Color Integration:** Integrate the color along the ray to produce the final pixel color $C(o, v)$ [S1]:
    $$ C(o, v) = \int_0^{+\infty} w_r(t) c(p(t), v) dt $$
6.  **Optimization:** Minimize the distance between the rendered images $I'_k$ and ground-truth images $I_k$.
7.  **Normal Regularization:** To resolve surface orientation ambiguity inherent in UDFs (where in/out tests are invalid for open surfaces), apply a normal regularization strategy. This enhances gradient consistency by leveraging normal information in the surface neighborhood [S1].
8.  **Importance Sampling:** Use a customized importance sampling strategy to accommodate the proposed weighting function $w_r(t)$ and ensure high-quality reconstruction of non-watertight surfaces [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def udensity_cdf(d, r):
    """
    Computes the Udensity CDF value for unsigned distance d.
    Must satisfy: cdf(0)=0, cdf(inf)=1, cdf'(d)>0, cdf''(d)<0 for d>0.
    The specific function form is not provided in sources; 
    this is a placeholder for a valid function satisfying constraints [S1].
    """
    # Example implementation satisfying constraints (not from source, [G])
    # Using a sigmoid-like shape scaled to [0, 1]
    # c'(d) > 0 and c''(d) < 0 implies concave increasing.
    # A simple concave function mapping [0, inf) to [0, 1) is 1 - exp(-r*d)
    # However, limit at inf must be 1. 
    # Let's use a generic placeholder that satisfies the math properties.
    # Note: The source does not specify the exact formula for varsigma_r, 
    # only its properties. We assume a valid implementation exists.
    return 1.0 - np.exp(-r * d)

def udensity_pdf(d, r):
    """
    Computes the Udensity PDF (derivative of CDF).
    """
    # Derivative of 1 - exp(-r*d) is r * exp(-r*d)
    # This is > 0 for d >= 0, r > 0.
    return r * np.exp(-r * d)

def compute_opaque_density(d, d_prev, varsigma_r_val, varsigma_r_prev_val):
    """
    Computes tau_r(t) = | (d/dt (varsigma_r(d(t)))) / varsigma_r(d(t)) |
    Using finite difference approximation for d/dt along the ray.
    """
    # d/dt (varsigma_r(d(t))) approx (varsigma_r(d(t)) - varsigma_r(d(t-dt))) / dt
    # Here d_prev is d(t-dt), varsigma_r_prev_val is varsigma_r(d_prev)
    # dt is assumed to be 1 for normalized step or handled by caller.
    # The source formula uses partial derivative w.r.t t.
    
    # Numerator: derivative of CDF along ray
    # d(CDF)/dt = d(CDF)/dd * dd/dt
    # We approximate dd/dt as (d - d_prev) / dt_step
    # And d(CDF)/dd as PDF(d)
    
    # However, the source formula is:
    # tau = | (d/dt (varsigma_r o Psi o p)) / (varsigma_r o Psi o p) |
    
    # Let's implement the chain rule explicitly as per source [S1]
    # d/dt (varsigma_r(d(t))) = varsigma_r'(d(t)) * d'(t)
    # varsigma_r'(d) is the PDF.
    
    pdf_d = udensity_pdf(d, r)
    pdf_d_prev = udensity_pdf(d_prev, r)
    
    # Approximate d'(t) = (d - d_prev) / dt
    # If dt is not provided, we assume uniform sampling or pass it.
    # For the purpose of the formula structure:
    # tau = | (pdf_d * d_dot) / varsigma_r(d) |
    
    # Since the source does not give the sampling step size, 
    # we assume the derivative is computed relative to the ray parameter t.
    # In practice, this requires the distance between samples.
    # We will assume a helper `get_dt()` exists or is passed.
    pass 

def render_neudf(ray_origin, ray_direction, network, num_samples, r):
    """
    Renders a single ray using NeUDF.
    """
    # Sample points along the ray
    # t values from 0 to +inf. In practice, bounded by near/far planes.
    t_values = np.linspace(0, 10.0, num_samples) # Placeholder bounds [G]
    
    # Sample positions p(t) = o + t * v
    positions = ray_origin + t_values[:, np.newaxis] * ray_direction
    
    # Query network for d and c
    # psi(v, x)
    d_values, color_values = network(ray_direction, positions)
    
    # Compute Udensity CDF values
    # varsigma_r(d)
    cdf_values = udensity_cdf(d_values, r)
    
    # Handle division by zero at d=0 where cdf=0
    # Add small epsilon for stability [G]
    epsilon = 1e-8
    cdf_values_safe = np.maximum(cdf_values, epsilon)
    
    # Compute opaque density tau_r(t)
    # tau_r(t) = | (d/dt (varsigma_r(d(t)))) / varsigma_r(d(t)) |
    # d/dt (varsigma_r(d(t))) = pdf(d(t)) * d'(t)
    
    # Compute PDF values
    pdf_values = udensity_pdf(d_values, r)
    
    # Compute derivative of d along ray: d'(t)
    # d'(t) = (d(t+dt) - d(t)) / dt
    # Using finite differences
    d_dot = np.zeros(num_samples)
    dt = t_values[1] - t_values[0]
    d_dot[:-1] = (d_values[1:] - d_values[:-1]) / dt
    d_dot[-1] = d_dot[-2] # Extrapolate last point
    
    # Numerator of tau
    numerator = pdf_values * d_dot
    
    # Tau = | numerator / cdf |
    tau = np.abs(numerator / cdf_values_safe)
    
    # Compute transmittance T(t) = exp(- integral_0^t tau(u) du)
    # Using cumulative trapezoidal integration
    tau_integral = np.cumsum(tau) * dt
    transmittance = np.exp(-tau_integral)
    
    # Compute weights w(t) = tau(t) * T(t)
    # Note: Standard volume rendering weight is tau(t) * T(t) * dt?
    # Source formula: w_r(t) = tau_r(t) e^{-int_0^t tau_r(u) du}
    # This w(t) is the weight for the integral C = int w(t) c(t) dt
    # So we multiply by dt in the final sum.
    weights = tau * transmittance
    
    # Render color
    # C = int w(t) c(t) dt
    rendered_color = np.sum(weights[:, np.newaxis] * color_values, axis=0) * dt
    
    return rendered_color

def train_step(images, cameras, network, r):
    """
    Training loop step.
    """
    # 1. Render images using render_neudf
    rendered_images = []
    for cam in cameras:
        # Cast rays for all pixels
        # ... (ray casting logic)
        # For each ray, call render_neudf
        pass
    
    # 2. Compute loss (MSE between rendered and ground truth)
    loss = compute_loss(rendered_images, images)
    
    # 3. Backpropagate
    # Includes normal regularization term [S1]
    # The normal regularization enhances gradient consistency 
    # by leveraging normal information in the surface neighborhood.
    # Normals can be derived from gradients of d w.r.t x.
    normal_reg_loss = compute_normal_regularization(network, cameras)
    total_loss = loss + normal_reg_loss
    
    # 4. Update network and parameter r
    update_parameters(network, r, total_loss)
```

**Parameters:**
*   $r$: Learnable parameter in the Udensity function $\varsigma_r(d)$ controlling the density distribution [S1].
*   $\epsilon$: Small threshold for defining the surface $\mathcal{S}_{\mathcal{O}}$ (mentioned in definition of $UDF_O$) [S1].

**Edge Cases and Pitfalls:**
*   **Spurious Surfaces:** Naively extending SDF-based weighting to UDF generates spurious surfaces because the SDF-based weighting function triggers undesirable local maxima in the void region [S1]. NeUDF avoids this via the Udensity formulation [S1].
*   **Open Surfaces:** Standard in/out tests are invalid for open surfaces. NeUDF resolves this ambiguity using normal regularization [S1].
*   **Boundary Conditions:** The Udensity function must satisfy $\varsigma_r(0)=0$ and $\lim_{d \to \infty} \varsigma_r(d)=1$. Failure to satisfy these leads to incorrect density accumulation [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples per ray and the depth of the neural network $\psi$. The integration requires evaluating the Udensity function and its derivative for each sample.
*   **Memory Complexity:** Dominated by the storage of the neural network parameters.
*   **Strengths:** Capable of reconstructing surfaces with arbitrary topologies, including open surfaces, which SDF-based methods cannot handle [S1]. Outperforms state-of-the-art methods on challenging datasets like DTU, MGN, and Deep Fashion 3D [S1].
*   **Limitations:** Requires multi-view 2D image supervision. The naive extension of SDF methods fails, necessitating the complex Udensity formulation [S1].
*   **When to use:** When reconstructing non-watertight or open-surface objects from multi-view images.
*   **When not to use:** When only closed surfaces are expected and SDF-based methods (like [Volume SDF Rendering](1344-volume-sdf-rendering.md)) are sufficient, as they may be simpler.

## Background
NeUDF was introduced in the paper "NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering" [S1]. It is the first attempt to reconstruct surfaces with arbitrary topologies solely from 2D image supervision [S1]. The method was evaluated on public datasets including DTU [21], MGN [5], Deep Fashion 3D [61], and BMVS [56]. It builds upon the differentiable volume rendering framework inspired by NeuS [53] [S1].

## Sources
[S1] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
