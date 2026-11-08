# Error Metric for Vector Fields

Also called: Ev(T), Vector Field Error Metric, Weighted Vector Field Error, Weighted Vector Field Error Metric

## Overview
The Error Metric for Vector Fields computes a data-dependent error bound for a vector field within a tetrahedron to guide view-dependent rendering decisions, such as adaptive mesh refinement or early ray termination. Unlike scalar field error metrics that rely on a single isovalue, this method combines errors from multiple vector field parameters—magnitude, gradient, curl, and divergence—using user-defined weights to provide a comprehensive measure of geometric and topological fidelity [S1]. It is distinct from sibling methods like [Domain Colouring](1833-domain-colouring.md) or [Stream Lines](4796-stream-lines.md) because it does not directly produce visual output but rather provides a quantitative metric for optimizing the sampling density of the vector field during rendering [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A trivariate vector field function $F(\mathbf{x}) = (F_1(\mathbf{x}), F_2(\mathbf{x}), F_3(\mathbf{x}))$ defined over a domain.
*   A tetrahedron $T$ with vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$ and known vector values at these vertices.
*   User-defined weights $a_1, a_2, a_3, a_4$ for magnitude, gradient, curl, and divergence respectively.
*   User-defined target values for magnitude $v_l$ and curl magnitude $c_l$ (used to define the specific error components).

**Outputs:**
*   A scalar error value $E_v(T)$ representing the weighted error of the vector field approximation within tetrahedron $T$.

### Data Structures
The method operates on the geometry of a single tetrahedron $T$.
*   **Vertices:** Four 3D points $\mathbf{v}_i \in \mathbb{R}^3$.
*   **Vector Values:** Four 3D vectors $\mathbf{f}_i \in \mathbb{R}^3$ corresponding to the field values at $\mathbf{v}_i$.
*   **Volume:** $|T|$, the scalar volume of the tetrahedron.
*   **Sample Points:** A set of points $\mathbf{x}$ distributed within $T$ for integration. The source notes that $\gamma_0, \gamma_1, \gamma_2$ terms can be precomputed and are independent of the specific isovalue/parameter, allowing efficient error computation [S1].

### Formulas and Definitions
The overall error $E_v(T)$ is a weighted sum of four individual error metrics:
$$ E_v(T) = a_1 E_l(T) + a_2 E_{\nabla}(T) + a_3 E_{curl}(T) + a_4 E_{div}(T) $$

Where:
*   $a_1, a_2, a_3, a_4$ are user-defined weights.
*   $E_l(T)$ is the error with respect to vector magnitude.
*   $E_{\nabla}(T)$ is the error with respect to the gradient field magnitude.
*   $E_{curl}(T)$ is the error with respect to the curl field magnitude.
*   $E_{div}(T)$ is the error with respect to the divergence field.

**1. Linear Interpolation**
Let $f(\mathbf{x})$ be the vector at point $\mathbf{x}$ linearly interpolated from the vertex values $\mathbf{f}_i$ of tetrahedron $T$.

**2. Vector Magnitude Error ($E_l$)**
This metric compares the error in the magnitude of the vector field against a user-specified target magnitude $v_l$. Let $g_{v_l}$ be a function related to the target magnitude $v_l$ (specifically, the source defines the term inside the square as the difference between the transformed exact magnitude and transformed interpolated magnitude) [S1]. The formula provided in the notes is:
$$ E_l(T) = \frac{1}{|T|} \sum_{\mathbf{x} \in T} \left( g_{v_l} - F(\mathbf{x}) - (g_{v_l} - f(\mathbf{x})) \right)^2 $$
*Note: The source text contains a typographical ambiguity in the notation $g_{v_l} - F(\mathbf{x})$. Based on the context of scalar error metrics and the structure of the formula, $g_{v_l}$ acts as a constant or function value associated with the target magnitude $v_l$. The term represents the squared difference between the error of the exact field magnitude relative to $v_l$ and the interpolated field magnitude relative to $v_l$. In standard implementations, this is often $\left( \|F(\mathbf{x})\| - v_l - (\|f(\mathbf{x})\| - v_l) \right)^2 = \left( \|F(\mathbf{x})\| - \|f(\mathbf{x})\| \right)^2$ if $g$ is identity, or a weighted distance. The source explicitly writes $g_{v_l} - F(\mathbf{x})$, which implies $g$ might be a scalar projection or the notation is shorthand for the magnitude difference. We follow the source's explicit formula structure.*

**3. Gradient Field ($\nabla F$)**
The gradient of the vector field is a tensor defined as:
$$ \nabla F = \left( \frac{\partial F}{\partial x}, \frac{\partial F}{\partial y}, \frac{\partial F}{\partial z} \right) $$
The error $E_{\nabla}(T)$ is defined analogously to $E_l(T)$ but using the magnitude of the gradient tensor.

**4. Curl Field ($\text{curl } F$)**
The curl measures the vorticity or "swirliness" of the vector field:
$$ \text{curl } F = \left( \frac{\partial F_3}{\partial y} - \frac{\partial F_2}{\partial z}, \frac{\partial F_1}{\partial z} - \frac{\partial F_3}{\partial x}, \frac{\partial F_2}{\partial x} - \frac{\partial F_1}{\partial y} \right) $$
The error $E_{curl}(T)$ is defined analogously to $E_l(T)$ using the magnitude of the curl vector.

**5. Divergence Field ($\text{div } F$)**
The divergence measures the rate of expansion per unit volume:
$$ \text{div } F = \frac{\partial F_1}{\partial x} + \frac{\partial F_2}{\partial y} + \frac{\partial F_3}{\partial z} $$
The error $E_{div}(T)$ is defined analogously to the scalar error metric $E_s(T)$ (which is not detailed in the provided notes but is standard for scalar fields) [S1].

### Parameters
*   **Weights ($a_1, a_2, a_3, a_4$):** User-defined weights balancing the importance of magnitude, gradient, curl, and divergence errors.
    *   *Typical Values:* In the tornado example, $a_2$ and $a_4$ were set to 0, focusing only on magnitude and curl [S1].
*   **Target Magnitude ($v_l$):** A specific vector magnitude chosen by the user for the $E_l$ calculation.
    *   *Example:* 0.15 in the tornado dataset [S1].
*   **Target Curl Magnitude ($c_l$):** A specific curl magnitude chosen by the user for the $E_{curl}$ calculation.
    *   *Example:* 0.15 in the tornado dataset [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_vector_field_error(T, F_exact, F_interpolated, a1, a2, a3, a4, v_l, c_l):
    """
    Computes the weighted error metric Ev(T) for a vector field over tetrahedron T.
    
    Args:
        T: Tetrahedron object with vertices and volume |T|.
        F_exact: Function returning exact vector F(x) at any point x.
        F_interpolated: Function returning linearly interpolated vector f(x) from T's vertices.
        a1, a2, a3, a4: Weights for magnitude, gradient, curl, divergence errors.
        v_l: User-defined target vector magnitude.
        c_l: User-defined target curl magnitude.
        
    Returns:
        Ev: Scalar error value.
    """
    # 1. Compute Magnitude Error El(T)
    # Note: The source formula uses a summation over sample points x in T.
    # In practice, this is an integral approximated by sampling.
    # The term g_vl - F(x) is interpreted as the difference in magnitude relative to v_l.
    # Based on the source notation: (g_vl - F(x) - (g_vl - f(x)))^2
    # This simplifies to (F(x) - f(x))^2 if g is linear/identity, but the source
    # explicitly ties it to v_l. We implement the magnitude difference squared.
    
    sum_mag_error = 0.0
    num_samples = 100 # Number of sample points in T for integration approximation
    for x in sample_points_in_tetrahedron(T, num_samples):
        F_x = F_exact(x)
        f_x = F_interpolated(x)
        
        # Magnitude of exact and interpolated vectors
        mag_F = magnitude(F_x)
        mag_f = magnitude(f_x)
        
        # Error term based on source formula structure
        # Source: (g_vl - F(x) - (g_vl - f(x)))^2
        # Interpreted as squared difference of magnitudes relative to target v_l
        # If g_vl is a constant offset, it cancels out, leaving (mag_F - mag_f)^2
        # However, if g_vl implies a specific projection or weighting, it must be applied.
        # Given the ambiguity, we use the magnitude difference squared as the core error.
        error_term = (mag_F - mag_f) ** 2
        sum_mag_error += error_term
        
    El = sum_mag_error / T.volume

    # 2. Compute Gradient Error E_nabla(T)
    # Analogous to El, but using the magnitude of the gradient tensor.
    # Gradient tensor magnitude is Frobenius norm of the 3x3 Jacobian.
    sum_grad_error = 0.0
    for x in sample_points_in_tetrahedron(T, num_samples):
        # Compute Jacobian J = dF/dx for exact and interpolated
        J_exact = compute_jacobian(F_exact, x)
        J_interp = compute_jacobian(F_interpolated, x)
        
        mag_J_exact = frobenius_norm(J_exact)
        mag_J_interp = frobenius_norm(J_interp)
        
        # Assuming target gradient magnitude is not explicitly weighted by a scalar v_l
        # in the same way, or it is 0. The source says "analogously to El".
        # If a target gradient magnitude existed, it would be subtracted.
        # Here we assume the error is simply the difference in gradient magnitudes.
        error_term = (mag_J_exact - mag_J_interp) ** 2
        sum_grad_error += error_term
        
    E_nabla = sum_grad_error / T.volume

    # 3. Compute Curl Error E_curl(T)
    # Analogous to El, using curl magnitude.
    sum_curl_error = 0.0
    for x in sample_points_in_tetrahedron(T, num_samples):
        curl_exact = compute_curl(F_exact, x)
        curl_interp = compute_curl(F_interpolated, x)
        
        mag_curl_exact = magnitude(curl_exact)
        mag_curl_interp = magnitude(curl_interp)
        
        # Error term
        error_term = (mag_curl_exact - mag_curl_interp) ** 2
        sum_curl_error += error_term
        
    E_curl = sum_curl_error / T.volume

    # 4. Compute Divergence Error E_div(T)
    # Analogous to scalar error Es(T).
    sum_div_error = 0.0
    for x in sample_points_in_tetrahedron(T, num_samples):
        div_exact = compute_divergence(F_exact, x)
        div_interp = compute_divergence(F_interpolated, x)
        
        # Scalar error typically squared difference
        error_term = (div_exact - div_interp) ** 2
        sum_div_error += error_term
        
    E_div = sum_div_error / T.volume

    # 5. Combine with weights
    Ev = a1 * El + a2 * E_nabla + a3 * E_curl + a4 * E_div
    
    return Ev

def sample_points_in_tetrahedron(T, n):
    """Generates n random points uniformly distributed within tetrahedron T."""
    # Barycentric coordinate sampling
    points = []
    for _ in range(n):
        r1 = random()
        r2 = random()
        r3 = random()
        # Ensure points are within the simplex
        if r1 + r2 + r3 > 1:
            r1 = 1 - r1
            r2 = 1 - r2
            r3 = 1 - r3
            s = r1 + r2 + r3
            r1 /= s
            r2 /= s
            r3 /= s
            
        # Interpolate using vertices v0, v1, v2, v3
        p = T.v0 * (1 - r1 - r2 - r3) + T.v1 * r1 + T.v2 * r2 + T.v3 * r3
        points.append(p)
    return points

def compute_jacobian(F, x):
    """Computes the 3x3 Jacobian matrix of vector field F at x using finite differences."""
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
    """Computes curl F at x using finite differences."""
    h = 1e-5
    F_x = F(x)
    
    # Partial derivatives needed for curl
    # curl F = (dF3/dy - dF2/dz, dF1/dz - dF3/dx, dF2/dx - dF1/dy)
    
    # dF/dy
    x_y = x.copy(); x_y[1] += h
    dF_dy = (F(x_y) - F_x) / h
    
    # dF/dz
    x_z = x.copy(); x_z[2] += h
    dF_dz = (F(x_z) - F_x) / h
    
    # dF/dx
    x_x = x.copy(); x_x[0] += h
    dF_dx = (F(x_x) - F_x) / h
    
    curl = [
        dF_dy[2] - dF_dz[1],
        dF_dz[0] - dF_dx[2],
        dF_dx[1] - dF_dy[0]
    ]
    return curl

def compute_divergence(F, x):
    """Computes div F at x using finite differences."""
    h = 1e-5
    F_x = F(x)
    
    # dF1/dx
    x_x = x.copy(); x_x[0] += h
    dF1_dx = (F(x_x)[0] - F_x[0]) / h
    
    # dF2/dy
    x_y = x.copy(); x_y[1] += h
    dF2_dy = (F(x_y)[1] - F_x[1]) / h
    
    # dF3/dz
    x_z = x.copy(); x_z[2] += h
    dF3_dz = (F(x_z)[2] - F_x[2]) / h
    
    return dF1_dx + dF2_dy + dF3_dz

def magnitude(v):
    return (v[0]**2 + v[1]**2 + v[2]**2)**0.5

def frobenius_norm(M):
    return (sum(M[i][j]**2 for i in range(3) for j in range(3)))**0.5
```

### Edge Cases and Pitfalls
*   **Zero Weights:** If weights $a_i$ are set to zero, the corresponding error component is ignored. In the tornado example, $a_2$ and $a_4$ were zero, simplifying the metric to magnitude and curl only [S1].
*   **Precision:** The computation involves derivatives (gradient, curl, divergence), which are sensitive to noise in the vector field data. Finite difference approximations require careful choice of step size $h$ [S1, G].
*   **Boundary Conditions:** The error metric is defined per tetrahedron. Boundary tetrahedra may have fewer neighbors or different interpolation properties, but the formula remains the same [S1].
*   **Sample Density:** The summation $\sum_{\mathbf{x} \in T}$ requires sufficient sampling to accurately approximate the integral. Insufficient samples may lead to inaccurate error estimates [S1].

## Operating characteristics
*   **Time Complexity:** The error computation for a single tetrahedron is $O(N \cdot D)$, where $N$ is the number of sample points per tetrahedron and $D$ is the cost of evaluating the field and its derivatives (typically constant for a fixed stencil). The total time for a volume is $O(T \cdot N \cdot D)$, where $T$ is the number of tetrahedra.
*   **Memory Complexity:** $O(T)$ to store the error values for each tetrahedron. The source notes that storing two additional values per tetrahedron (likely precomputed terms $\gamma_0, \gamma_1, \gamma_2$) reduces traversal time by up to 50% [S1].
*   **Performance:** The method allows for efficient view-dependent visualization by pruning tetrahedra with low error. The reduction in traversed tetrahedra depends heavily on the dataset and viewpoint [S1].
*   **Strengths:** Provides a comprehensive error metric for vector fields by combining multiple physical properties (magnitude, curl, divergence). Allows user control via weights.
*   **Limitations:** Requires computation of derivatives, which can be noisy. The choice of weights and target values ($v_l, c_l$) is subjective and dataset-dependent.

## Background
The Error Metric for Vector Fields was introduced in the context of adaptive multi-valued volume data visualization. It extends the scalar error metric to vector fields by incorporating multiple parameters (magnitude, gradient, curl, divergence) [S1]. The method was applied to a tornado-like dataset generated by Crawfis and Max to illustrate flow patterns in 3D flow fields [S1]. The underlying scalar error metric and its precomputation techniques ($\gamma$ terms) are described in the same source [S1].

## Sources
[S1] Adaptive Multi-Valued Volume Data Visualization using Data-Dependent Error Metrics, https://doi.org/10.2316/journal.205.2005.2.205-4291, fetched 2026-10-26
