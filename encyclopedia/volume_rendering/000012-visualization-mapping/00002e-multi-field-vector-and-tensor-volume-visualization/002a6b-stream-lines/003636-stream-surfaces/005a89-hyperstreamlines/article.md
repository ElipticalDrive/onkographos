# Hyperstreamlines

Also called: Tensor Streamlines

## Overview
Hyperstreamlines are a visualization technique for tensor fields that extends the concept of streamlines from vector fields by defining the longitudinal direction via an eigenvector and encoding the transverse tensor structure in the cross-section [S1, S2, unofficial]. Unlike standard streamlines which only show flow direction, hyperstreamlines provide a continuous representation of the local anisotropy and orientation of the tensor field, solving the problem of visualizing directional diffusion or stress in 3D space [S1, S2, unofficial]. They are distinct from [Stream Surfaces](7696-stream-surfaces.md) by explicitly using the tensor's eigen-decomposition to define the geometry of the primitive being swept, rather than stitching adjacent vector streamlines.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D tensor field $T(\mathbf{x})$ (typically symmetric positive definite, e.g., diffusion tensors in DT-MRI or stress tensors).
*   **Output:** A set of geometric primitives (tubes or ribbons) representing the hyperstreamlines.

**Data Structures:**
*   **Tensor Field:** A grid where each cell contains a $3 \times 3$ symmetric matrix.
*   **Eigen-decomposition:** For each point $\mathbf{x}$, compute eigenvalues $\lambda_1 \ge \lambda_2 \ge \lambda_3$ and corresponding orthonormal eigenvectors $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$.
*   **Primitive:** A geometric shape (ellipse or cross) defined in the local frame of the streamline.

**Steps:**
1.  **Select Longitudinal Eigenvector:** Choose one of the three eigenfields to define the direction of propagation. Commonly, the major eigenvector $\mathbf{v}_1$ (associated with $\lambda_1$) is used, but medium ($\mathbf{v}_2$) or minor ($\mathbf{v}_3$) eigenvectors can be selected to create "medium" or "minor" hyperstreamlines [S2, unofficial].
2.  **Integrate Streamline:** Compute the streamline curve $\mathbf{r}(t)$ by integrating the selected eigenvector field $\mathbf{v}_{long}$:
    $$ \frac{d\mathbf{r}}{dt} = \mathbf{v}_{long}(\mathbf{r}(t)) $$
    using a numerical integrator (e.g., Runge-Kutta) [S2, unofficial].
3.  **Define Cross-Section:** At each point $\mathbf{r}(t)$ along the streamline, the remaining two eigenvectors $\mathbf{v}_{trans1}$ and $\mathbf{v}_{trans2}$ define the orientation of the cross-section [S1, S2, unofficial].
4.  **Sweep Primitive:** Sweep a geometric primitive along the streamline. Two common primitives are defined in [S2, unofficial]:
    *   **Ellipse:** The major and minor axes of the ellipse are aligned with $\mathbf{v}_{trans1}$ and $\mathbf{v}_{trans2}$. The lengths of these axes are typically proportional to the square roots of the corresponding eigenvalues ($\sqrt{\lambda_{trans1}}, \sqrt{\lambda_{trans2}}$) or fixed constants. This results in a tubular shape [S2, unofficial].
    *   **Cross:** Two line segments intersecting at the streamline center, aligned with $\mathbf{v}_{trans1}$ and $\mathbf{v}_{trans2}$. The length of the arms is controlled by the transverse eigenvalues. This results in a helical shape if the transverse eigenvectors rotate along the path [S2, unofficial].
5.  **Coloring:** Color the hyperstreamline based on the longitudinal eigenvector or the scalar value of the longitudinal eigenvalue [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hyperstreamline(tensor_field, start_point, primitive_type='ellipse'):
    """
    Computes a single hyperstreamline.
    tensor_field: Function T(x) returning 3x3 matrix.
    start_point: Initial 3D vector.
    primitive_type: 'ellipse' or 'cross'.
    """
    # 1. Select major eigenvector as longitudinal direction
    # In practice, one might select v1, v2, or v3. Here we assume v1.
    
    current_pos = start_point
    path_points = [current_pos]
    
    # Integration parameters
    dt = 0.01
    max_steps = 1000
    
    for i in range(max_steps):
        # Get tensor at current position
        T = tensor_field(current_pos)
        
        # Eigen-decomposition
        eigenvalues, eigenvectors = eigen_decompose(T)
        # Sort eigenvalues: lambda1 >= lambda2 >= lambda3
        # Corresponding eigenvectors: v1, v2, v3
        
        # Longitudinal direction (major eigenvector)
        v_long = eigenvectors[:, 0] # v1
        
        # Transverse directions
        v_trans1 = eigenvectors[:, 1] # v2
        v_trans2 = eigenvectors[:, 2] # v3
        
        # Integrate step
        next_pos = current_pos + dt * v_long
        
        # Store cross-section data for rendering
        cross_section = {
            'position': current_pos,
            'v_long': v_long,
            'v_trans1': v_trans1,
            'v_trans2': v_trans2,
            'lambda1': eigenvalues[0],
            'lambda2': eigenvalues[1],
            'lambda3': eigenvalues[2]
        }
        path_points.append(cross_section)
        
        current_pos = next_pos
        
        # Check for termination (e.g., leaving domain or critical point)
        if not is_in_domain(current_pos):
            break
            
    return path_points

def render_cross_section(cross_section, primitive_type):
    """
    Generates geometry for the cross-section at a specific point.
    """
    v1 = cross_section['v_trans1']
    v2 = cross_section['v_trans2']
    l1 = cross_section['lambda2'] # Using lambda2 for transverse axis 1
    l2 = cross_section['lambda3'] # Using lambda3 for transverse axis 2
    
    if primitive_type == 'ellipse':
        # Create ellipse in local frame defined by v1, v2
        # Major axis length proportional to sqrt(l1), minor to sqrt(l2)
        axis_a = sqrt(l1)
        axis_b = sqrt(l2)
        return create_ellipse(center=cross_section['position'], 
                              axis1=v1 * axis_a, 
                              axis2=v2 * axis_b)
    elif primitive_type == 'cross':
        # Create cross with arms along v1 and v2
        arm_len = sqrt(l1) # Example scaling
        arm1 = v1 * arm_len
        arm2 = v2 * arm_len
        return create_cross(center=cross_section['position'], 
                            arm1=arm1, 
                            arm2=arm2)
```

**Edge Cases and Pitfalls:**
*   **Isotropic Regions:** In regions where eigenvalues are nearly equal (isotropic), the eigenvectors are undefined or unstable due to noise and partial voluming effects, leading to ambiguous or erratic hyperstreamline orientations [S1, unofficial].
*   **Critical Points:**
    *   **Locus:** Points where the longitudinal eigenvector vanishes (e.g., $\lambda_1 \to 0$ or direction becomes undefined).
    *   **Surface:** Points where the cross-section becomes singular (e.g., $\lambda_2 \to 0$ or $\lambda_3 \to 0$, reducing the ellipse to a line or point) [S2, unofficial].
*   **Helical Artifacts:** When using the cross primitive, the transverse eigenvectors may rotate significantly along the streamline, causing the cross arms to twist and form a helical shape, which may be visually confusing [S2, unofficial].

## Subtopics

### Tensorlines
A variation that stabilizes the propagation direction by blending the major eigenvector with the input velocity vector, reducing ambiguity in isotropic regions [S1, unofficial].

### Streamtubes
Used in linear anisotropic regions, often implying a simpler tubular representation without the explicit cross-sectional encoding of transverse eigenvectors [S1, unofficial].

### Streamsurfaces
Used in planar anisotropic surfaces, representing a 2D manifold of flow rather than a 1D line with a cross-section [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the eigen-decomposition at each integration step. For a streamline of length $L$ with step size $h$, complexity is $O(L/h \cdot C_{eigen})$, where $C_{eigen}$ is the cost of $3 \times 3$ eigen-decomposition (constant time).
*   **Memory Complexity:** $O(N)$ for storing the path points and cross-section data, where $N$ is the number of integration steps.
*   **Strengths:** Provides a rich, continuous visualization of tensor orientation and anisotropy; intuitive mapping of eigenvectors to geometry.
*   **Limitations:** Computationally expensive due to repeated eigen-decompositions; sensitive to noise in isotropic regions; visual clutter in dense fields.
*   **When to use:** When the primary interest is the direction of maximum diffusion/stress and the local anisotropy structure (e.g., fiber tracking in DT-MRI).
*   **When not to use:** In highly isotropic regions where eigenvectors are unstable; when computational resources are limited.

## Background
Hyperstreamlines were introduced by Delmarcelle and Hesselink in 1992/1993 as an extension of streamlines to tensor fields [S1, unofficial]. They are widely used in the visualization of diffusion tensor MRI (DT-MRI) data to visualize neural pathways, where the major eigenvector represents the direction of axonal fibers [S2, unofficial]. The technique addresses the limitations of scalar or vector-only visualizations by incorporating the full tensor information into the geometry of the visualization primitive.

## Sources
[S1] PDF Tensor Field Visualization - ETH Z, https://cgl.ethz.ch/teaching/scivis_common/stuff/StuttgartCourse/VIS-Modules-08-Tensor_Field_Visualization.pdf, fetched 2026-10-28 (unofficial)
[S2] PDF Tensor Visualization, http://www.csc.lsu.edu/~karki/courses/SV/19-TensorVis.pdf, fetched 2026-10-28 (unofficial)
