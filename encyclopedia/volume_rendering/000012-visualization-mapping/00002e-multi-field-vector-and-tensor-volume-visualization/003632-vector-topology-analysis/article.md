# Vector Topology Analysis

Also called: Critical Point Classification, Topology skeleton tracing, 临界点分类

## Overview
Vector Topology Analysis is a technique to analyze vector field topology based on critical point classification to capture global features of a flow field [S1, unofficial]. Critical points are defined as points in a vector field where the vector is zero [S2, unofficial]. It sets itself apart from advection-based methods like Stream Lines by focusing on geometric singularities rather than continuous trajectories, producing a compact topology skeleton [S1, unofficial]. Unlike scalar field topology methods, it operates directly on vector components to determine flow behavior near stagnation points [G].

## How it works
**Inputs:** A discrete vector field $\mathbf{v}(\mathbf{x})$ defined on a grid or mesh [G].
**Outputs:** A set of classified critical points and a topology skeleton [S1, unofficial].

**Data Structures:**
1. **Vector Field Grid:** A 3D array storing vector components $(v_x, v_y, v_z)$ at each cell center or vertex [G].
2. **Critical Point List:** A list of points $\mathbf{p}_i$ with associated classification labels and eigenvector data [G].
3. **Topology Skeleton:** A set of line segments or polylines representing separatrices [G].

**Steps:**

1. **Locate Critical Points:** Identify points $\mathbf{x}$ where $\mathbf{v}(\mathbf{x}) = \mathbf{0}$ [S1, unofficial]. In 2D fields, solve the simultaneous bilinear equations analytically [S1, unofficial]. In 3D fields, use numerical root-finding methods such as Newton-Raphson to locate zeros [S1, unofficial].
2. **Compute Jacobian:** For each critical point $\mathbf{p}$, compute the Jacobian matrix $J$ of the vector field at that point [G].
   $$J(\mathbf{p}) = \begin{bmatrix} \frac{\partial v_x}{\partial x} & \frac{\partial v_x}{\partial y} & \frac{\partial v_x}{\partial z} \\ \frac{\partial v_y}{\partial x} & \frac{\partial v_y}{\partial y} & \frac{\partial v_y}{\partial z} \\ \frac{\partial v_z}{\partial x} & \frac{\partial v_z}{\partial y} & \frac{\partial v_z}{\partial z} \end{bmatrix}_{\mathbf{x}=\mathbf{p}}$$
   where $\frac{\partial v_i}{\partial x_j}$ denotes the partial derivative of the $i$-th vector component with respect to the $j$-th spatial coordinate [G].
3. **Classify Critical Points:** Compute the eigenvalues $\lambda_1, \lambda_2, \lambda_3$ and eigenvectors of $J(\mathbf{p})$ [S1, unofficial]. Classify the point based on the signs and nature of the eigenvalues into one of the following types [S1, S2, unofficial]: Repelling Focus, Saddle Point, Repelling Node, Attracting Focus, Center (Vortex Center), or Attracting Node [S1, unofficial]. Source [S2, unofficial] also refers to these categories as Source, Sink, Saddle, and Center.
4. **Trace Topology Skeleton:** Create the skeleton by tracing streamlines starting from the neighboring region of the critical point and following the direction of eigenvectors [S1, unofficial]. Specifically, separatrices are traced along eigenvectors corresponding to contracting or expanding directions [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def vector_topology_analysis(vector_field, grid_bounds, step=0.1, tol=1e-6, max_iter=50):
    """
    Analyzes vector field topology by finding and classifying critical points.
    Args:
        vector_field: 3D array of shape (Nx, Ny, Nz, 3) containing (vx, vy, vz).
        grid_bounds: Tuple of (x_min, x_max, y_min, y_max, z_min, z_max).
        step: Initial step size for root finding.
        tol: Convergence tolerance for zero magnitude.
        max_iter: Maximum Newton-Raphson iterations.
    Returns:
        critical_points: List of dicts with 'position', 'type', 'eigenvalues', 'eigenvectors'.
        skeleton: List of line segments connecting critical points.
    """
    critical_points = []
    skeleton = []
    
    # Step 1: Locate Critical Points
    # Sources do not specify the exact 3D root-finding implementation [G].
    zero_points = find_critical_points_newton(vector_field, grid_bounds, step, tol, max_iter)
    
    for p in zero_points:
        # Step 2: Compute Jacobian at critical point p
        J = compute_jacobian_central_diff(vector_field, p)
        
        # Step 3: Classify based on eigenvalues
        eigenvalues, eigenvectors = compute_eigenvalues_eigenvectors(J)
        point_type = classify_point(eigenvalues)
        
        critical_points.append({
            "position": p,
            "type": point_type,
            "eigenvalues": eigenvalues,
            "eigenvectors": eigenvectors
        })
        
        # Step 4: Trace Topology Skeleton
        separatrices = trace_separatrices(p, eigenvectors, vector_field)
        skeleton.extend(separatrices)
        
    return critical_points, skeleton

def find_critical_points_newton(field, bounds, step, tol, max_iter):
    """Standard Newton-Raphson solver for 3D vector field zeros [G]."""
    zeros = []
    x_min, x_max, y_min, y_max, z_min, z_max = bounds
    # Grid scan to find initial guesses near zero crossings
    for x in range(0, field.shape[0]-1, 2):
        for y in range(0, field.shape[1]-1, 2):
            for z in range(0, field.shape[2]-1, 2):
                p = [x, y, z]
                for _ in range(max_iter):
                    v = field[p[0], p[1], p[2], :]
                    if math.sqrt(sum(c**2 for c in v)) < tol:
                        zeros.append(p)
                        break
                    J = compute_jacobian_central_diff(field, p)
                    # Solve J * delta = -v for delta, then p = p + delta
                    # Standard linear algebra step [G]
                    delta = solve_linear_system(J, [-c for c in v])
                    p = [p[i] + delta[i] for i in range(3)]
    return zeros

def compute_jacobian_central_diff(field, p):
    """Computes the 3x3 Jacobian matrix using central differences [G]."""
    dx, dy, dz = 1.0, 1.0, 1.0
    J = [[0.0]*3 for _ in range(3)]
    for i in range(3):
        for j in range(3):
            coord = [p[0], p[1], p[2]]
            coord[j] += 0.5
            p_plus = field[max(0, min(coord[0], field.shape[0]-1)), 
                           max(0, min(coord[1], field.shape[1]-1)), 
                           max(0, min(coord[2], field.shape[2]-1)), i]
            coord[j] -= 1.0
            p_minus = field[max(0, min(coord[0], field.shape[0]-1)), 
                            max(0, min(coord[1], field.shape[1]-1)), 
                            max(0, min(coord[2], field.shape[2]-1)), i]
            J[i][j] = (p_plus - p_minus) / 1.0
    return J

def compute_eigenvalues_eigenvectors(J):
    """Computes eigenvalues and eigenvectors for a 3x3 matrix [G]."""
    # Standard characteristic polynomial solver [G]
    # Returns eigenvalues (list of 3 floats) and eigenvectors (list of 3 lists)
    pass # Placeholder for standard linear algebra routine [G]

def classify_point(eigenvalues):
    """Classifies critical point based on eigenvalue signs [S1, S2, unofficial]."""
    real_parts = [e.real if hasattr(e, 'real') else e for e in eigenvalues]
    has_positive = any(r > 1e-6 for r in real_parts)
    has_negative = any(r < -1e-6 for r in real_parts)
    if has_positive and has_negative:
        return "Saddle Point"
    elif all(r > 1e-6 for r in real_parts):
        return "Repelling Node"
    elif all(r < -1e-6 for r in real_parts):
        return "Attracting Node"
    elif any(abs(r) < 1e-6 for r in real_parts):
        return "Center"
    else:
        return "Repelling Focus" if has_positive else "Attracting Focus"

def trace_separatrices(start_point, eigenvectors, field):
    """Traces streamlines along eigenvector directions [S1, S3, unofficial]."""
    lines = []
    for ev in eigenvectors:
        line = [start_point]
        p = list(start_point)
        for _ in range(100):
            v = field[max(0, min(p[0], field.shape[0]-1)), 
                      max(0, min(p[1], field.shape[1]-1)), 
                      max(0, min(p[2], field.shape[2]-1)), :]
            norm_v = math.sqrt(sum(c**2 for c in v))
            if norm_v < 1e-6:
                break
            direction = [ev[i] / norm_v for i in range(3)]
            p = [p[i] + direction[i] for i in range(3)]
            line.append(p)
        lines.append(line)
    return lines
```

**Parameters:**
*   **Numerical Integration Order:** High-order adaptive schemes are recommended for accurate trajectory tracing, especially near high-gradient regions [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Numerical Integration Failure:** Precise streamlines connecting critical points are difficult to compute using numerical integration because critical points usually exist in regions with high gradients and discontinuity, where most numerical methods fail to produce accurate results [S1, unofficial].
*   **Incomplete Feature Capture:** Flow topology based on critical point theory does not necessarily exhaust interesting flow phenomena; scientists may want to know flow directions in regions surrounding objects where no critical points exist [S1, unofficial].
*   **Discrete Field Issues:** In discrete scalar/vector fields, the Morse function condition may not hold, requiring pre-processing to enforce topological consistency [S3, unofficial].

## Subtopics

### Separation and attachment line extraction
Identifies locations on field boundaries where flow separates from or attaches to the boundary, serving as starting points for separatrix integration [S3, unofficial]. The method projects the flow field onto the tessellated boundary mesh of the data set, computes the Jacobian $J$ of the projected flow field $v$ on the boundary $\partial \Omega$, and finds positions $p$ where the projected flow direction is orthogonal to the major or minor eigenvector of the Jacobian: $v(p)|_{\partial \Omega} \cdot e_{\{max,min\}}(J(p)|_{\partial \Omega}) = 0$ [S3, unofficial]. Lines are approximated by segments created from intersections with edges of boundary mesh elements [S3, unofficial]. Accuracy depends on boundary mesh tessellation density and projection fidelity [S3, unofficial].

### Vortex Detection
Methods used to identify and extract vortex structures (rotational structures of fluid) from complex flow fields [S2, unofficial]. Common criteria include the Q-criterion and Lambda2 [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides a compact, global representation of the flow field; effective for identifying major flow structures like vortices and stagnation zones [S1, S2, unofficial].
*   **Limitations:** Computationally expensive for high-resolution 3D fields; sensitive to noise and discretization errors; flow topology based on critical point theory does not necessarily exhaust interesting flow phenomena [S1, unofficial].
*   **When to use:** When the primary goal is to understand the global topology and major flow structures rather than detailed local trajectories [S1, unofficial].
*   **When not to use:** When high-precision trajectory tracking in high-gradient regions is required, or when the field lacks significant critical points [S1, unofficial].

## Background
The technique is based on the work of Helman and Hesslink [29] (cited in [S1, unofficial]), who proposed analyzing vector field topology through critical point classification to understand solution trajectories of ordinary differential equations [S1, unofficial]. It is widely used in computational fluid dynamics (CFD) visualization to help scientists quickly locate interesting regions such as vortices and stagnation points [S2, unofficial]. The classification scheme is standard in dynamical systems theory applied to flow visualization [S1, S2, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] 第9 章：科学可视化, http://simingchen.me/smart-textbook/ch9/index.html, fetched 2026-10-30 (unofficial)
[S3] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
