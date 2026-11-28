# Vector Topology Analysis

Also called: Critical point classification, Topology skeleton tracing

## Overview
Vector Topology Analysis is a technique for analyzing vector field topology based on critical point classification to capture global features of a flow field [S1, unofficial]. It identifies points of zero vector magnitude, classifies their local behavior via Jacobian eigenvalues, and constructs a skeleton of streamlines to reveal the underlying structure of the flow [S1, unofficial]. It differs from advection techniques like Stream Lines [4796-stream-lines.md] by focusing on the geometric skeleton of singularities rather than tracing arbitrary paths, and from glyph methods like Line Glyphs [2569-line-glyphs-for-vector-field-visualization.md] by providing a global topological summary rather than local directional samples.

## How it works
**Inputs:** A 2D or 3D vector field $\mathbf{V}(\mathbf{x}) = (u, v, w)$ defined on a grid or unstructured mesh.
**Outputs:** A set of classified critical points and a topology skeleton (streamlines) connecting them.

**Data Structures:**
1.  **Critical Point List:** A list of points $\mathbf{p}_i$ with coordinates $(x, y, z)$ and classification type.
2.  **Jacobian Matrix:** For each critical point, a $3 \times 3$ matrix $J$ containing partial derivatives $\frac{\partial V_j}{\partial x_k}$.
3.  **Eigenvectors:** Three unit vectors $\mathbf{e}_1, \mathbf{e}_2, \mathbf{e}_3$ corresponding to the eigenvalues of $J$.

**Steps:**

1.  **Identify Critical Points:**
    Find all points $\mathbf{x}$ where the vector magnitude is zero:
    $$ \mathbf{V}(\mathbf{x}) = \mathbf{0} $$
    *   **In 2D:** Solve the simultaneous bilinear equations for $u(x,y)=0$ and $v(x,y)=0$ analytically [S1, unofficial].
    *   **In 3D:** Use numerical methods, such as Newton-Raphson, to locate roots where $\mathbf{V}(\mathbf{x}) \approx \mathbf{0}$ [S1, unofficial].

2.  **Compute Jacobian and Eigenvalues:**
    At each critical point $\mathbf{p}$, compute the Jacobian matrix $J$:
    $$ J = \begin{bmatrix} \frac{\partial u}{\partial x} & \frac{\partial u}{\partial y} & \frac{\partial u}{\partial z} \\ \frac{\partial v}{\partial x} & \frac{\partial v}{\partial y} & \frac{\partial v}{\partial z} \\ \frac{\partial w}{\partial x} & \frac{\partial w}{\partial y} & \frac{\partial w}{\partial z} \end{bmatrix} $$
    Compute the eigenvalues $\lambda_1, \lambda_2, \lambda_3$ and their corresponding eigenvectors $\mathbf{e}_1, \mathbf{e}_2, \mathbf{e}_3$ of $J$ [S1, unofficial].

3.  **Classify Critical Points:**
    Classify the point based on the signs of the eigenvalues into one of the following types [S1, unofficial]:
    *   **Repelling Node:** All $\lambda_i > 0$.
    *   **Attracting Node:** All $\lambda_i < 0$.
    *   **Repelling Focus:** Complex eigenvalues with positive real part.
    *   **Attracting Focus:** Complex eigenvalues with negative real part.
    *   **Saddle Point:** Mixed signs of real eigenvalues.
    *   **Center:** Purely imaginary eigenvalues (or zero real parts in specific contexts).

4.  **Trace Topology Skeleton:**
    Create a topology skeleton by tracing streamlines starting from the neighboring region of the critical point and following the direction of the eigenvectors [S1, unofficial].
    *   Initialize integration at a small distance $\epsilon$ from the critical point along the direction of an eigenvector.
    *   Integrate the vector field $\mathbf{V}$ to generate the streamline.

**Pitfalls and Edge Cases:**
*   **Numerical Integration Failure:** Precise streamlines connecting critical points are difficult to compute using numerical integration because critical points usually exist in regions with high gradients and discontinuity, where most numerical methods fail to produce accurate results [S1, unofficial].
*   **Incomplete Coverage:** Flow topology based on critical point theory does not necessarily exhaust interesting flow phenomena; scientists may want to know flow directions in regions surrounding objects where no critical points exist [S1, unofficial].

## Operating characteristics
*   **Complexity:** Finding critical points in 3D requires iterative numerical solvers (Newton-Raphson), making it computationally expensive per point. Classification is $O(1)$ per point after Jacobian computation. Skeleton tracing is $O(N)$ where $N$ is the number of integration steps.
*   **Strengths:** Effective in capturing global features of a flow field [S1, unofficial]. Provides a compact representation of the flow structure.
*   **Limitations:** Fails to represent flow features in regions without critical points. Numerical accuracy degrades near high-gradient regions associated with critical points.
*   **When to use:** When the goal is to understand the global topological structure and singularities of a steady-state flow field.
*   **When not to use:** When detailed flow directions in non-singular regions are required, or when the field is highly unsteady (topology changes over time).

## Background
Helman and Hesslink [29] proposed analyzing vector field topology based on critical point classification, a method widely used to analyze solution trajectories of ordinary differential equations [S1, unofficial]. This technique is categorized as a topological technique for flow visualization, distinct from advection (streamlines) and texture-based methods [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
