# Boundary Path Integral Formulation

Also called: Boundary Path Integral Formulation for Differentiable Rendering, Interior-to-Boundary Reparameterization, Path Integral Gradient Estimation, Reparameterized Boundary Path Integral, Volume Boundary Path Integral Formulation, Zhang et al. 2020 formulation, Zhang et al. 2020 reparameterization, Zhang et al. method

## Overview
The Boundary Path Integral Formulation is a method for estimating the derivatives of the path integral in differentiable volume rendering with respect to object shape parameters [S1, S3, unofficial]. It solves the problem of computing gradients when scene geometry depends on parameters $\theta$, by separating the differentiation of the path integral into interior and boundary terms [S1, unofficial]. Unlike its siblings [Reparameterized Volume Sampling](3309-reparameterized-volume-sampling.md) and [Reparameterized Monte Carlo Volume Rendering](4798-reparameterized-monte-carlo-volume-rendering.md), which typically rely on differentiable sampling within the volume or standard CDF inversion, this method converts interior volume integrals into boundary area integrals using the divergence theorem [S2]. This eliminates the need to evaluate the deformation mapping inside the volume, avoiding expensive tetrahedralizations [S2].

## How it works
The method reparameterizes the rendering integral using a reference configuration independent of the shape parameter $\theta$, then applies the divergence theorem to move derivative terms from the volume interior to the material boundary [S2][S3, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Scene parameter $\theta \in \mathbb{R}$ (scalar for simplicity, generalizable to $\mathbb{R}^m$) [S2].
    *   Reference configuration $\mathcal{B}$ (typically $\mathcal{B} = \mathcal{M}(\theta_0)$ when estimating gradient at $\theta_0$) [S3, unofficial].
    *   Mapping $\mathbf{X}(\cdot, \theta): \mathcal{B} \to \mathcal{M}(\theta)$, a differentiable one-to-one mapping from reference to current geometry [S3, unofficial].
    *   Measurement contribution function $\hat{f}(\vec{p})$ in material space [S3, unofficial].
*   **Outputs:**
    *   Gradient estimate $\frac{dI}{d\theta}$ [S3, unofficial].

### Data Structures
*   **Material Path $\vec{p}$:** A sequence of vertices $\vec{p} = (p_0, \dots, p_N)$ in the reference space $\hat{\Omega}$ [S3, unofficial].
*   **Velocity Field $\mathbf{v}(\mathbf{p})$:** Defined as the derivative of the mapping with respect to $\theta$: $\mathbf{v}(\mathbf{p}) := d_\theta \mathbf{X}(\mathbf{p}, \theta)$ [S2].
*   **Normal Velocity $v_\perp(\mathbf{p}_K)$:** The component of $\mathbf{v}(\mathbf{p}_K)$ perpendicular to the boundary $\partial \mathcal{B}_V$ at vertex $p_K$ [S2].

### Assumptions
1.  **S.1:** Translucent objects contain no refractive or reflective boundaries (null BSDFs) and there are no visibility boundaries (shadows) dependent on $\theta$ [S2]. This ensures no visibility discontinuities and continuity of the material measurement contribution in the interior [S2].
2.  **S.2:** $\theta$ controls only shapes, not material properties (extinction/scattering) [S2].
3.  **Reference Configuration:** $\mathcal{B}_V = \mathcal{V}(0)$ and $\mathcal{B}_M = \mathcal{M}(0)$. The mapping $\mathbf{X}(\cdot, 0)$ is the identity, and Jacobians $J(\mathbf{p}, 0) = 1$ [S2].

### Formulas
The forward intensity is $I = \int_{\Omega} f(\vec{x}) \, d\mu(\vec{x})$ [S3, unofficial]. Reparameterizing to material space $\hat{\Omega}$ yields $I = \int_{\hat{\Omega}} \hat{f}(\vec{p}) \, d\mu(\vec{p})$ [S3, unofficial].

The gradient is decomposed into interior and boundary terms [S3, unofficial]:
$$
\frac{dI}{d\theta} = \int_{\hat{\Omega}} \frac{d\hat{f}(\vec{p})}{d\theta} d\mu(\vec{p}) + \int_{\partial \hat{\Omega}} \hat{f}^B \hat{f}^S \hat{f}^D d\mu
$$
The first term is the standard interior derivative. The second term is the boundary term, unique to differentiable rendering [S3, unofficial].

The interior term $\int_{\hat{\Omega}} d_\theta \hat{f}(\vec{p}) d\mu(\vec{p})$ is converted to a boundary integral using the divergence theorem [S2]. For a function $h$ (where $\hat{f}$ is composed of such terms), the derivative is:
$$
d_\theta \left( h(\bar{\mathbf{x}}) \prod_{n=0}^N J(\mathbf{p}_n, \Theta) \right) = \sum_{n=0}^N \left( (\nabla_n h)(\bar{\mathbf{p}}) \cdot \mathbf{v}(\mathbf{p}_n) + h(\bar{\mathbf{p}}) \partial_\Theta J(\mathbf{p}_n, \Theta) \right)
$$
Applying the divergence theorem to the volume integral of this sum over a volume segment $\mathcal{B}_V$ at index $K$:
$$
\int_{\mathcal{B}_V} \left( (\nabla h)(\bar{\mathbf{p}}) \cdot \mathbf{v}(\mathbf{p}_n) + h(\bar{\mathbf{p}}) \partial_\theta J(\mathbf{p}_n, \theta) \right) d\mathbf{V}(\mathbf{p}_n) = \int_{\partial\mathcal{B}_V} h(\bar{\mathbf{p}}) v_\perp(\mathbf{p}_n) d\mathbf{A}(\mathbf{p}_n)
$$
Summing over all possible boundary vertices $K$ ($0 < K < N$), the total interior derivative becomes a boundary path integral [S2]:
$$
\int_{\hat{\Omega}} d_\theta \hat{f}(\vec{p}) d\mu(\vec{p}) = \int_{\hat{\Omega}} \text{interior } f(\vec{p}) d\mu(\vec{p}) + \int_{\partial \hat{\Omega}} f(\vec{p}) v_\perp(\mathbf{p}_K) d\dot{\mu}(\vec{p})
$$
where $\partial \hat{\Omega}$ is the volume boundary path space containing exactly one volume boundary vertex $p_K$ on $\partial \mathcal{B}_V$ [S2]. The measure $d\dot{\mu}$ is defined as [S2]:
$$
d\dot{\mu}(\vec{p}) = dA(\mathbf{p}_K) \prod_{n \neq K} \begin{cases} dV(\mathbf{p}_n), & (\mathbf{p}_n \in \mathcal{B}_Y) \\ dA(\mathbf{p}_n), & (\mathbf{p}_n \in \mathcal{B}_M) \end{cases}
$$
Here $\mathcal{B}_Y$ is the volume boundary and $\mathcal{B}_M$ is the surface boundary [S2].

### Steps
1.  **Define Reference:** Set reference volume $\mathcal{B}_V$ and surface $\mathcal{B}_M$ to the current geometry at $\theta=0$. Set $\mathbf{X}(\cdot, 0)$ to identity [S2].
2.  **Sample Paths:** Generate material paths $\vec{p}$ in $\hat{\Omega}$ using standard path tracing or BDPT [S3, unofficial].
3.  **Compute Interior Term:** For each path, compute the standard derivative of the material measurement contribution $\frac{d\hat{f}}{d\theta}$ with respect to $\theta$ [S3, unofficial]. This involves differentiating transmittance and scattering terms that depend on $\theta$ [S2].
4.  **Compute Boundary Term:**
    *   Identify if the path has a vertex $p_K$ on the volume boundary $\partial \mathcal{B}_V$ with $0 < K < N$ [S2].
    *   If yes, compute the normal velocity $v_\perp(\mathbf{p}_K)$ [S2].
    *   Evaluate the path contribution $f(\vec{p})$ at the boundary vertex [S2].
    *   Accumulate $f(\vec{p}) v_\perp(\mathbf{p}_K)$ into the boundary integral estimate [S2].
5.  **Estimate Gradient:** Sum the Monte Carlo estimates of the interior term and the boundary term [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_boundary_path_integral(scene, theta, num_samples):
    """
    Estimates dI/dtheta using the Boundary Path Integral Formulation.
    Assumes theta=0 and reference configuration matches current geometry.
    """
    interior_sum = 0.0
    boundary_sum = 0.0
    
    for _ in range(num_samples):
        # 1. Sample a material path p_bar in reference space
        p_bar = sample_material_path(scene.reference_config)
        
        # 2. Map to ordinary space x_bar = X(p_bar, theta)
        # At theta=0, X is identity, so x_bar = p_bar
        x_bar = map_to_ordinary_space(p_bar, theta)
        
        # 3. Compute standard measurement contribution f(x_bar)
        # This includes transmittance, scattering, BSDFs, etc.
        f_val = compute_measurement_contribution(x_bar, scene)
        
        # 4. Compute Interior Term: d(f_hat)/dtheta
        # This requires differentiating the integrand with respect to theta.
        # For translucent objects with null BSDFs, this captures
        # changes in transmittance due to shape deformation.
        # Note: This term is non-zero because transmittance depends on path length
        # which changes with shape deformation.
        interior_derivative = compute_interior_derivative(p_bar, scene, theta)
        interior_sum += interior_derivative / num_samples
        
        # 5. Compute Boundary Term
        # Check if path has exactly one vertex p_K on the volume boundary
        # with index K such that 0 < K < N.
        boundary_vertex_index = find_boundary_vertex_index(p_bar, scene.reference_config)
        
        if boundary_vertex_index is not None:
            K = boundary_vertex_index
            p_K = p_bar[K]
            
            # Compute normal velocity v_perp at p_K
            # v(p) = d_theta X(p, theta)
            # v_perp is the component perpendicular to the boundary normal
            v = compute_velocity(p_K, theta)
            n = compute_boundary_normal(p_K, scene.reference_config)
            v_perp = dot(v, n)
            
            # The boundary term integrand is f(p_bar) * v_perp
            # Note: f(p_bar) here refers to the measurement contribution
            # evaluated at the mapped path, but the measure change accounts
            # for the boundary constraint.
            boundary_contribution = f_val * v_perp
            boundary_sum += boundary_contribution / num_samples
            
    return interior_sum + boundary_sum

def compute_interior_derivative(p_bar, scene, theta):
    """
    Computes the derivative of the material measurement contribution
    with respect to theta. This involves chain rule application on
    transmittance and scattering terms.
    """
    # d/dtheta (f_hat) = sum over vertices of (grad_n f . v_n + f * dJ/dtheta)
    # At theta=0, J=1, but dJ/dtheta is non-zero.
    # For simplicity in this pseudocode, we assume a helper computes
    # the total derivative of the path weight product.
    return compute_path_weight_derivative(p_bar, scene, theta)

def find_boundary_vertex_index(p_bar, ref_config):
    """
    Returns the index K of the first vertex on the volume boundary
    such that 0 < K < N. Returns None if no such vertex or multiple.
    """
    N = len(p_bar) - 1
    boundary_indices = []
    for i in range(1, N): # 0 < K < N
        if is_on_volume_boundary(p_bar[i], ref_config):
            boundary_indices.append(i)
    
    if len(boundary_indices) == 1:
        return boundary_indices[0]
    else:
        return None
```

### Parameters
*   **$\theta$:** Shape parameter. Derivatives evaluated at $\theta=0$ [S2].
*   **Reference Configuration $\mathcal{B}$:** Typically $\mathcal{M}(\theta_0)$ [S3, unofficial].

### Edge Cases and Pitfalls
*   **Visibility Discontinuities:** If Assumption S.1 is violated (refractive/reflective boundaries or $\theta$-dependent shadows), the velocity of visibility boundaries is nontrivial to compute without tetrahedralization [S2].
*   **Multiple Boundary Vertices:** Paths with two or more vertices on the boundary have zero measure and contribute nothing to the first-order derivative [S2].
*   **Computational Cost:** Gradient computation for a single image takes seconds to tens of seconds, making it impractical for neural network training [S1, unofficial].
*   **Tetrahedralization:** Previous methods required tetrahedralization to interpolate mapping $X$ into the volume interior. This method avoids that overhead [S2].

## Subtopics

### Unidirectional Estimator
Samples paths from camera to light. Simple to implement but high variance in participating media. [Link: [Reparameterization Estimators](6499-reparameterization-estimators.md)]

### Bidirectional Estimator
Samples paths from camera and light, connecting them. Better for complex lighting but more complex to implement with boundary terms. [Link: [Reparameterization Estimators](6499-reparameterization-estimators.md)]

### Multiple Importance Sampling (MIS)
Combines unidirectional and bidirectional estimates to reduce variance. Requires careful weighting of boundary terms. [Link: [Reparameterization Estimators](6499-reparameterization-estimators.md)]

## Operating characteristics
*   **Time Complexity:** High. Gradient computation for a single image takes seconds to tens of seconds [S1, unofficial].
*   **Memory Complexity:** Depends on path storage. Avoids storing tetrahedral meshes, reducing overhead compared to previous methods [S2].
*   **Accuracy:** Unbiased [S1, unofficial]. Validated against finite-difference gradients [S2].
*   **Strengths:** Does not require explicit silhouette edge finding [S1, unofficial]. Avoids expensive tetrahedralizations [S2]. Handles shape derivatives efficiently for translucent objects [S2].
*   **Limitations:** Slow for iterative optimization (e.g., neural networks) [S1, unofficial]. Assumes no refractive/reflective boundaries or $\theta$-dependent shadows [S2].
*   **When to Use:** Inverse rendering of translucent objects where shape optimization is needed and computational time per iteration is not critical.
*   **When Not to Use:** Real-time differentiable rendering, scenes with complex refractive interfaces, or when training deep neural networks requiring many gradient steps.

## Background
The Boundary Path Integral Formulation was introduced by Zhang et al. in 2020 [S1, S3, unofficial]. It extends the differential path integral framework to handle volume rendering with respect to shape parameters [S2]. Prior methods used tetrahedral meshes to interpolate deformation mappings into the volume interior, which was computationally expensive [S2]. This method applies the divergence theorem to convert interior volume integrals into boundary area integrals, eliminating the need for interior mapping evaluations [S2]. It is part of the broader class of reparameterization estimators for differentiable rendering [S1, unofficial].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S2] PDF Efficient Path-Space Differentiable Volume Rendering With Respect To Shapes, https://www.shuangz.com/projects/psdr-vol2-egsr23/psdr-vol2-egsr23.pdf, fetched 2026-10-27
[S3] Variance-aware Differentiable Rendering, https://core.ac.uk/works/337282320, fetched 2026-11-02 (unofficial)
