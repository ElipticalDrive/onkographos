# Error Metric for Vector Fields

Also called: Ev(T), Vector Field Error Metric, Weighted Vector Field Error, Weighted Vector Field Error Metric

## Overview
The Error Metric for Vector Fields computes a data-dependent error bound for a vector field within a tetrahedron to guide view-dependent rendering decisions [S1]. Unlike scalar field error metrics that rely on a single isovalue, this method combines errors from multiple vector field parameters—magnitude, gradient, curl, and divergence—using user-defined weights to provide a comprehensive measure of geometric and topological fidelity [S1]. It is distinct from sibling methods like [Domain Colouring](1833-domain-colouring.md) or [Stream Lines](4796-stream-lines.md) because it provides a quantitative metric for optimizing the sampling density of the vector field during rendering [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A trivariate vector field function $F(\mathbf{x}) = (F_1(\mathbf{x}), F_2(\mathbf{x}), F_3(\mathbf{x}))$ defined over a domain [S1].
*   A tetrahedron $T$ with vertices and known vector values at these vertices [S1].
*   User-defined weights $a_1, a_2, a_3, a_4$ for magnitude, gradient, curl, and divergence respectively [S1].
*   A specific vector magnitude $v_l$ chosen by the user [S1].
*   A function $g$ from the scalar error metric [S1].

**Outputs:**
*   A scalar error value $E_v(T)$ representing the weighted error of the vector field approximation within tetrahedron $T$ [S1].

### Data Structures
The method operates on the geometry of a single tetrahedron $T$.
*   **Vertices:** Four 3D points $\mathbf{v}_i \in \mathbb{R}^3$.
*   **Vector Values:** Four 3D vectors $\mathbf{f}_i \in \mathbb{R}^3$ corresponding to the field values at $\mathbf{v}_i$.
*   **Volume:** $|T|$, the scalar volume of the tetrahedron.
*   **Sample Points:** A set of points $\mathbf{x}$ distributed within $T$ for integration [S1].

### Formulas and Definitions
The overall error $E_v(T)$ is a weighted sum of four individual error metrics:
$$ E_v(T) = a_1 E_l(T) + a_2 E_{\nabla}(T) + a_3 E_{curl}(T) + a_4 E_{div}(T) $$
Where:
*   $a_1, a_2, a_3, a_4$ are user-defined weights [S1].
*   $E_l(T)$ is the error with respect to vector magnitude [S1].
*   $E_{\nabla}(T)$ is the error with respect to the gradient field magnitude [S1].
*   $E_{curl}(T)$ is the error with respect to the curl field magnitude [S1].
*   $E_{div}(T)$ is the error with respect to the divergence field [S1].

**1. Linear Interpolation**
Let $f(\mathbf{x})$ be the vector at point $\mathbf{x}$ linearly interpolated from the vertex values $\mathbf{f}_i$ of tetrahedron $T$ [S1].

**2. Vector Magnitude Error ($E_l$)**
This metric compares the error in the magnitude of the vector field against a user-specified target magnitude $v_l$ [S1]. The formula is:
$$ E_l(T) = \frac{1}{|T|} \sum_{\mathbf{x} \in T} \left( g(v_l - F(\mathbf{x})) - g(v_l - f(\mathbf{x})) \right)^2 $$
where $g$ is the same function as in the scalar metric [S1].

**3. Gradient Field ($\nabla F$)**
The gradient of the vector field is a tensor defined as:
$$ \nabla F = \left( \frac{\partial F}{\partial x}, \frac{\partial F}{\partial y}, \frac{\partial F}{\partial z} \right) $$
The error $E_{\nabla}(T)$ is defined analogously to $E_l(T)$ but using the magnitude of the gradient tensor [S1].

**4. Curl Field ($\text{curl } F$)**
The curl measures the vorticity or "swirliness" of the vector field:
$$ \text{curl } F = \left( \frac{\partial F_3}{\partial y} - \frac{\partial F_2}{\partial z}, \frac{\partial F_1}{\partial z} - \frac{\partial F_3}{\partial x}, \frac{\partial F_2}{\partial x} - \frac{\partial F_1}{\partial y} \right) $$
The error $E_{curl}(T)$ is defined analogously to $E_l(T)$ using the magnitude of the curl vector [S1].

**5. Divergence Field ($\text{div } F$)**
The divergence measures the rate of expansion per unit volume:
$$ \text{div } F = \frac{\partial F_1}{\partial x} + \frac{\partial F_2}{\partial y} + \frac{\partial F_3}{\partial z} $$
The error $E_{div}(T)$ is defined analogously to the scalar error metric $E_s(T)$ [S1].

### Parameters
*   **Weights ($a_1, a_2, a_3, a_4$):** User-defined weights balancing the importance of magnitude, gradient, curl, and divergence errors [S1]. In the tornado example, $a_2$ and $a_4$ were set to zero [S1].
*   **Target Magnitude ($v_l$):** A specific vector magnitude chosen by the user for the $E_l$ calculation [S1]. In the tornado dataset, $v_l = 0.15$ [S1].
*   **Target Curl Magnitude:** A specific curl magnitude chosen by the user for the $E_{curl}$ calculation [S1]. In the tornado dataset, the curl magnitude was $0.15$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

def compute_vector_field_error(T, F_exact, f_interpolated, a1, a2, a3, a4, v_l, g):
    """
    Computes the weighted error metric Ev(T) for a vector field over tetrahedron T.
    """
    # 1. Compute Magnitude Error El(T)
    sum_mag_error = 0.0
    num_samples = 100 # Number of sample points in T for integration approximation [G]
    for x in sample_points_in_tetrahedron(T, num_samples):
        F_x = F_exact(x)
        f_x = f_interpolated(x)
        # Source formula: (g(v_l - F(x)) - g(v_l - f(x)))^2 [S1]
        # Note: Source notation implies g operates on the scalar difference of magnitudes [G]
        error_term = (g(v_l - magnitude(F_x)) - g(v_l - magnitude(f_x))) ** 2
        sum_mag_error += error_term
    El = sum_mag_error / T.volume

    # 2. Compute Gradient Error E_nabla(T)
    # Analogous to El, using magnitude of gradient tensor [S1]
    sum_grad_error = 0.0
    for x in sample_points_in_tetrahedron(T, num_samples):
        J_exact = compute_jacobian(F_exact, x) # Finite differences [G]
        J_interp = compute_jacobian(f_interpolated, x) # Finite differences [G]
        mag_J_exact = frobenius_norm(J_exact)
        mag_J_interp = frobenius_norm(J_interp)
        error_term = (g(mag_J_exact) - g(mag_J_interp)) ** 2
        sum_grad_error += error_term
    E_nabla = sum_grad_error / T.volume

    # 3. Compute Curl Error E_curl(T)
    # Analogous to El, using magnitude of curl vector [S1]
    sum_curl_error = 0.0
    for x in sample_points_in_tetrahedron(T, num_samples):
        curl_exact = compute_curl(F_exact, x) # Finite differences [G]
        curl_interp = compute_curl(f_interpolated, x) # Finite differences [G]
        mag_curl_exact = magnitude(curl_exact)
        mag_curl_interp = magnitude(curl_interp)
        error_term = (g(mag_curl_exact) - g(mag_curl_interp)) ** 2
        sum_curl_error += error_term
    E_curl = sum_curl_error / T.volume

    # 4. Compute Divergence Error E_div(T)
    # Analogous to scalar error Es(T) [S1]
    sum_div_error = 0.0
    for x in sample_points_in_tetrahedron(T, num_samples):
        div_exact = compute_divergence(F_exact, x) # Finite differences [G]
        div_interp = compute_divergence(f_interpolated, x) # Finite differences [G]
        error_term = (g(div_exact) - g(div_interp)) ** 2
        sum_div_error += error_term
    E_div = sum_div_error / T.volume

    # 5. Combine with weights
    Ev = a1 * El + a2 * E_nabla + a3 * E_curl + a4 * E_div
    return Ev

def sample_points_in_tetrahedron(T, n):
    """Generates n points uniformly distributed within tetrahedron T using barycentric coordinates [G]."""
    points = []
    v0, v1, v2, v3 = T.vertices
    for _ in range(n):
        r1 = random.random()
        r2 = random.random()
        r3 = random.random()
        if r1 + r2 + r3 > 1:
            r1 = 1 - r1
            r2 = 1 - r2
            r3 = 1 - r3
            s = r1 + r2 + r3
            r1 /= s
            r2 /= s
            r3 /= s
        p = v0 * (1 - r1 - r2 - r3) + v1 * r1 + v2 * r2 + v3 * r3
        points.append(p)
    return points

def compute_jacobian(F, x):
    """Computes the 3x3 Jacobian matrix of vector field F at x using finite differences [G]."""
    h = 1e-5
    F_x = F(x)
    J = [[0.0]*3 for _ in range(3)]
    for i in range(3):
        x_plus = x.copy()
        x_plus[i] += h
        F_x_plus = F(x_plus)
        for j in range(3):
            J[j][i] = (F_x_plus[j] - F_x[j]) / h
    return J

def compute_curl(F, x):
    """Computes curl F at x using finite differences [G]."""
    h = 1e-5
    F_x = F(x)
    x_x = x.copy(); x_x[0] += h
    x_y = x.copy(); x_y[1] += h
    x_z = x.copy(); x_z[2] += h
    dF_dx = (F(x_x) - F_x) / h
    dF_dy = (F(x_y) - F_x) / h
    dF_dz = (F(x_z) - F_x) / h
    return [dF_dy[2] - dF_dz[1], dF_dz[0] - dF_dx[2], dF_dx[1] - dF_dy[0]]

def compute_divergence(F, x):
    """Computes div F at x using finite differences [G]."""
    h = 1e-5
    F_x = F(x)
    x_x = x.copy(); x_x[0] += h
    x_y = x.copy(); x_y[1] += h
    x_z = x.copy(); x_z[2] += h
    dF1_dx = (F(x_x)[0] - F_x[0]) / h
    dF2_dy = (F(x_y)[1] - F_x[1]) / h
    dF3_dz = (F(x_z)[2] - F_x[2]) / h
    return dF1_dx + dF2_dy + dF3_dz

def magnitude(v):
    return (v[0]**2 + v[1]**2 + v[2]**2)**0.5

def frobenius_norm(M):
    return (sum(M[i][j]**2 for i in range(3) for j in range(3)))**0.5
```

### Edge Cases and Pitfalls
*   **Zero Weights:** If weights $a_i$ are set to zero, the corresponding error component is ignored. In the tornado example, $a_2$ and $a_4$ were zero, simplifying the metric to magnitude and curl only [S1].
*   **Derivative Sensitivity:** The computation involves derivatives (gradient, curl, divergence), which are sensitive to noise in the vector field data. Finite difference approximations require careful choice of step size $h$ [G].
*   **Sample Density:** The summation $\sum_{\mathbf{x} \in T}$ requires sufficient sampling to accurately approximate the integral. Insufficient samples may lead to inaccurate error estimates [G].

## Operating characteristics
*   **Time Complexity:** The error computation for a single tetrahedron is $O(N \cdot D)$, where $N$ is the number of sample points per tetrahedron and $D$ is the cost of evaluating the field and its derivatives (typically constant for a fixed stencil) [G]. The total time for a volume is $O(T \cdot N \cdot D)$, where $T$ is the number of tetrahedra [G].
*   **Memory Complexity:** $O(T)$ to store the error values for each tetrahedron [G]. The source notes that storing two additional values per tetrahedron reduces traversal time by up to 50% [S1].
*   **Performance:** The method allows for efficient view-dependent visualization by pruning tetrahedra with low error. The reduction in traversed tetrahedra depends heavily on the dataset and viewpoint [S1].
*   **Strengths:** Provides a comprehensive error metric for vector fields by combining multiple physical properties (magnitude, curl, divergence). Allows user control via weights [S1].
*   **Limitations:** Requires computation of derivatives, which can be noisy. The choice of weights and target values ($v_l$) is subjective and dataset-dependent [S1].

## Background
The Error Metric for Vector Fields was introduced in the context of adaptive multi-valued volume data visualization. It extends the scalar error metric to vector fields by incorporating multiple parameters (magnitude, gradient, curl, divergence) [S1]. The method was applied to a tornado-like dataset generated by Crawfis and Max to illustrate flow patterns in 3D flow fields [S1]. The underlying scalar error metric and its precomputation techniques are described in the same source [S1].

## Sources
[S1] Adaptive Multi-Valued Volume Data Visualization using Data-Dependent Error Metrics, https://doi.org/10.2316/journal.205.2005.2.205-4291, fetched 2026-10-26
