# Point Calculus Faceted Solid Raycasting

Also called: Faceted Solid Intersection in Point Calculus, Point Calculus Ray Marching, Ray-Solid Intersection via Point Calculus Edge Cases, Tetrahedron/Prism Raycasting via Point Calculus, Visualization of Geometric Models of Faceted Solids in Point Calculus

## Overview
A GPU-accelerated raycasting method for visualizing faceted solids (e.g., tetrahedrons, prisms) defined as a three-parameter set of points in 3D space using point calculus [S1]. It determines ray-solid intersections by solving systems of equations via the Newton method, specifically by fixing edge parameters of the solid's parametric equation to 0 or 1 [S1]. Unlike standard analytic distance functions or triangle rasterization, this approach aims for a complete absence of visual artifacts on faceted solids by treating the solid as a continuous parametric volume [S1]. It is distinct from [Plücker-based Tetrahedral Cell Intersection](1767-pl-cker-based-tetrahedral-cell-intersection.md) by using iterative numerical solving of parametric systems rather than algebraic line-plane intersections, and from [Point-in-Tetrahedron Test via Barycentric Coordinates](2591-point-in-tetrahedron-test-via-barycentric-coordinates.md) by computing entry/exit intervals directly through parametric edge constraints rather than discrete containment tests [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Solid definition: Four vertices of a 3D simplex $A, B, C, D$ (coordinates $\mathbf{x}_A, \dots, \mathbf{x}_D$) and parametric functions $p(u,v,w), q(u,v,w), r(u,v,w), s(u,v,w)$ such that $p+q+r+s=1$ [S1]. For faceted solids like tetrahedrons, these are linear: $M = Auv\bar{w} + B\bar{v}\bar{w} + C\bar{u}\bar{v}\bar{w} + Dw$ where $\bar{u}=1-u$, etc. [S1].
    *   Camera parameters: Eye position $P$, target $T$, and screen pixel coordinates $(p_x, p_y)$ normalized to $[0,1]$ [S1].
    *   Constants: Maximum tracing distance $d_{max}=100$, numerical step $\Delta=10^{-3}$, max Newton iterations $N_{max}=100$ [S1].
*   **Outputs:**
    *   Color of the pixel, determined by whether the ray segment $PQ$ lies inside the solid and Lambertian shading [S1].

**Data Structures**
*   **Simplex Vertices:** Array of 4 vectors $\{A, B, C, D\}$, each 3 floats.
*   **Ray Segment:** Two vectors $P$ (origin) and $Q$ (end).
*   **Newton State:** Vector $\sigma = \{x, y, z\}$ representing the current approximation of the unknown parameters $(u, v, w, t)$ mapped to 3 variables [S1].

**Algorithm Steps**

1.  **Ray Generation:**
    Compute the ray direction basis vectors $\vec{k}, \vec{i}, \vec{j}$ from the camera position $P$ and target $T$ [S1]:
    $$ \vec{k} = \frac{T - P}{|T - P|} $$
    $$ \vec{i} = \frac{\{0,1,0\} \times \vec{k}}{|\{0,1,0\} \times \vec{k}|} $$
    $$ \vec{j} = \vec{i} \times \vec{k} $$
    Calculate the ray endpoint $Q$ using pixel coordinates $p_x, p_y$ and max distance $d_{max}$ [S1]:
    $$ Q = P + \frac{\vec{i} \cdot p_x + \vec{j} \cdot p_y + \vec{k}}{|\vec{i} \cdot p_x + \vec{j} \cdot p_y + \vec{k}|} d_{max} $$

2.  **Barycentric Coordinate Calculation (Tetrahedron Specific):**
    For a tetrahedron defined by simplex $ABCD$, calculate the barycentric coordinates for points $P$ and $Q$. First, compute the volume of the simplex $V_{ABCD}$ using the determinant [S1]:
    $$ V_{ABCD} = \det \begin{pmatrix} x_A - x_D & y_A - y_D & z_A - z_D \\ x_B - x_D & y_B - y_D & z_B - z_D \\ x_C - x_D & y_C - y_D & z_C - z_D \end{pmatrix} $$
    Compute volumes for sub-tetrahedra (e.g., $V_{PBCD}$) similarly by replacing the first vertex with $P$ or $Q$. The barycentric coordinates for $P$ are [S1]:
    $$ p_P = \frac{V_{PBCD}}{V_{ABCD}}, \quad q_P = \frac{V_{APCD}}{V_{ABCD}}, \quad r_P = \frac{V_{ABPD}}{V_{ABCD}}, \quad s_P = 1 - p_P - q_P - r_P $$
    Repeat for $Q$ to get $p_Q, q_Q, r_Q, s_Q$. Note: The factor $1/6$ in volume calculation is omitted as it cancels out in the ratio [S1].

3.  **Intersection Interval Determination (Tetrahedron Specific):**
    Determine the parameter $\lambda$ along the segment $PQ$ where the ray intersects the faces. The intersection point $S$ on face $BCD$ (where $p=0$) is found at [S1]:
    $$ \lambda = \frac{p_P}{p_P - p_Q} $$
    The point $S$ is [S1]:
    $$ S = P \frac{p_Q}{p_Q - p_P} + Q \frac{p_P}{p_P - p_Q} $$
    Similarly, calculate $\lambda$ for other faces. The ray segment lies inside the tetrahedron if $\lambda$ falls within the interval defined by the entry and exit faces [S1]:
    $$ \lambda \in \left[ \frac{r_P}{r_P - r_Q}; \frac{p_P}{p_P - p_Q} \right] $$
    If $\lambda$ is outside this interval, the ray does not intersect the solid volume [S1].

4.  **General Faceted Solid Intersection (Newton Method):**
    For general faceted solids defined by $M = A p + B q + C r + D s$, equate the ray equation to the solid equation [S1]:
    $$ P\bar{t} + Qt = A p(u,v,w) + B q(u,v,w) + C r(u,v,w) + D s(u,v,w) $$
    This yields a system of 3 scalar equations (x, y, z) with 4 unknowns $(u, v, w, t)$ [S1]. To solve, fix one parameter to an edge value (0 or 1) for each of the 6 systems corresponding to the solid's shells [S1]:
    1.  $u=0, x=v, y=w, z=t$
    2.  $u=1, x=v, y=w, z=t$
    3.  $v=0, x=u, y=w, z=t$
    4.  $v=1, x=u, y=w, z=t$
    5.  $w=0, x=u, y=v, z=t$
    6.  $w=1, x=u, y=v, z=t$
    
    For each system, solve for the remaining 3 unknowns using Newton's method [S1]:
    $$ \sigma^{(i)} = \sigma^{(i-1)} - J(\sigma^{(i-1)})^{-1} F(\sigma^{(i-1)}) $$
    where $\sigma = \{x, y, z\}$ maps to the specific unknowns for that system, and $t$ is always mapped to $z$ [S1].
    
    **Jacobian Calculation:**
    Compute the Jacobian matrix $J$ numerically using central differences with step $\Delta = 10^{-3}$ [S1]:
    $$ \frac{\partial F}{\partial x}(\sigma) = \frac{F(\{x + \Delta, y, z\}) - F(\{x - \Delta, y, z\})}{2\Delta} $$
    The full Jacobian is [S1]:
    $$ J(\sigma) = \begin{pmatrix} \left( \frac{\partial F}{\partial x}(\sigma) \right)_x & \left( \frac{\partial F}{\partial y}(\sigma) \right)_x & \left( \frac{\partial F}{\partial z}(\sigma) \right)_x \\ \left( \frac{\partial F}{\partial x}(\sigma) \right)_y & \left( \frac{\partial F}{\partial y}(\sigma) \right)_y & \left( \frac{\partial F}{\partial z}(\sigma) \right)_y \\ \left( \frac{\partial F}{\partial x}(\sigma) \right)_z & \left( \frac{\partial F}{\partial y}(\sigma) \right)_z & \left( \frac{\partial F}{\partial z}(\sigma) \right)_z \end{pmatrix} $$
    
    **Initialization and Degeneracy Handling:**
    Initialize $\sigma^{(0)} = \{0, 0, 0\}$ [S1]. Check if $J(\sigma^{(0)})$ is degenerate. If so, substitute the identity matrix for $J$ to correct the first approximation [S1].
    
    **Stopping Conditions:**
    1.  Accuracy: $\sqrt{(x^{(i)} - x^{(i-1)})^2 + (y^{(i)} - y^{(i-1)})^2 + (z^{(i)} - z^{(i-1)})^2} < \Delta$ [S1].
    2.  Max Iterations: $i > 100$ [S1].

5.  **Solution Validation:**
    For each of the 6 systems, check if the resulting parameters $(u, v, w, t)$ are all within the range $[0, 1]$ [S1]. Discard solutions outside this range.
    *   If two valid solutions exist with $t \in [0, 1]$, the ray passes through the solid. The segment between the corresponding $t$ values is inside [S1].
    *   If one valid solution exists, the ray touches the solid [S1].
    *   If no valid solutions exist, there is no intersection [S1].

6.  **Shading:**
    Calculate the normal vector $\vec{n}$ for Lambertian shading using the gradient of the distance function $\delta$ (distance to the solid on segment $PQ$) via finite differences [S1]:
    $$ \vec{d} = \{\delta(P + \vec{i} \cdot \Delta, Q) - \delta(P, Q), \delta(P + \vec{j} \cdot \Delta, Q) - \delta(P, Q), \delta(P + \vec{k} \cdot \Delta, Q) - \delta(P, Q)\} $$
    $$ \vec{n} = \frac{\vec{d}}{|\vec{d}|} $$
    Apply lighting model using $\vec{n}$ and light direction [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_calculus_faceted_solid_raycasting(A, B, C, D, p_func, q_func, r_func, s_func, P_eye, T_target, p_x, p_y, d_max=100.0, delta=1e-3, max_iter=100):
    # 1. Ray Generation
    k = normalize(T_target - P_eye)
    i = normalize(cross(vec3(0, 1, 0), k))
    j = cross(i, k)
    ray_dir = normalize(i * p_x + j * p_y + k)
    Q_end = P_eye + ray_dir * d_max
    
    # 2. Tetrahedron Specific Check (if solid is tetrahedron)
    # Calculate volumes
    V_ABCD = det(vec3(A.x-D.x, A.y-D.y, A.z-D.z), 
                 vec3(B.x-D.x, B.y-D.y, B.z-D.z), 
                 vec3(C.x-D.x, C.y-D.y, C.z-D.z))
    
    # Helper to get barycentric coords
    def get_barycentric(P_point, A_pt, B_pt, C_pt, D_pt, V_ABCD):
        V_PBCD = det(vec3(P_point.x-D_pt.x, P_point.y-D_pt.y, P_point.z-D_pt.z),
                     vec3(B_pt.x-D_pt.x, B_pt.y-D_pt.y, B_pt.z-D_pt.z),
                     vec3(C_pt.x-D_pt.x, C_pt.y-D_pt.y, C_pt.z-D_pt.z))
        V_APCD = det(vec3(A_pt.x-P_point.x, A_pt.y-P_point.y, A_pt.z-P_point.z),
                     vec3(C_pt.x-D_pt.x, C_pt.y-D_pt.y, C_pt.z-D_pt.z),
                     vec3(B_pt.x-D_pt.x, B_pt.y-D_pt.y, B_pt.z-D_pt.z)) # Note: order matters for sign
        V_ABPD = det(vec3(A_pt.x-D_pt.x, A_pt.y-D_pt.y, A_pt.z-D_pt.z),
                     vec3(B_pt.x-P_point.x, B_pt.y-P_point.y, B_pt.z-P_point.z),
                     vec3(C_pt.x-D_pt.x, C_pt.y-D_pt.y, C_pt.z-D_pt.z))
        
        p = V_PBCD / V_ABCD
        q = V_APCD / V_ABCD
        r = V_ABPD / V_ABCD
        s = 1.0 - p - q - r
        return p, q, r, s

    p_P, q_P, r_P, s_P = get_barycentric(P_eye, A, B, C, D, V_ABCD)
    p_Q, q_Q, r_Q, s_Q = get_barycentric(Q_end, A, B, C, D, V_ABCD)
    
    # Check intersection interval for tetrahedron
    # Interval [r_P/(r_P - r_Q), p_P/(p_P - p_Q)]
    # Note: This assumes specific face ordering. General case uses Newton below.
    # For general implementation, proceed to Newton method for robustness across solids.
    
    # 3. General Faceted Solid Intersection via Newton Method
    valid_intersections = []
    
    # Define the 6 systems
    systems = [
        {'fixed': ('u', 0), 'vars': ('v', 'w', 't')},
        {'fixed': ('u', 1), 'vars': ('v', 'w', 't')},
        {'fixed': ('v', 0), 'vars': ('u', 'w', 't')},
        {'fixed': ('v', 1), 'vars': ('u', 'w', 't')},
        {'fixed': ('w', 0), 'vars': ('u', 'v', 't')},
        {'fixed': ('w', 1), 'vars': ('u', 'v', 't')}
    ]
    
    for sys in systems:
        fixed_param, fixed_val = sys['fixed']
        x_var, y_var, z_var = sys['vars']
        
        # Initial guess
        sigma = vec3(0.0, 0.0, 0.0)
        
        for i in range(max_iter):
            # Map sigma to u, v, w, t
            u, v, w, t = map_sigma_to_params(sigma, fixed_param, fixed_val, x_var, y_var, z_var)
            
            # Evaluate F(sigma) = Ray(u,v,w,t) - Solid(u,v,w,t)
            # Ray point: P + t*(Q-P)
            ray_point = P_eye + t * (Q_end - P_eye)
            
            # Solid point
            p_val = p_func(u, v, w)
            q_val = q_func(u, v, w)
            r_val = r_func(u, v, w)
            s_val = s_func(u, v, w)
            solid_point = A * p_val + B * q_val + C * r_val + D * s_val
            
            F_val = ray_point - solid_point
            
            # Check convergence
            if length(F_val) < delta: # Simplified convergence check based on residual
                # Check if parameters are in [0,1]
                if 0 <= u <= 1 and 0 <= v <= 1 and 0 <= w <= 1 and 0 <= t <= 1:
                    valid_intersections.append(t)
                break
            
            # Compute Jacobian numerically
            J = compute_jacobian_numerical(sigma, fixed_param, fixed_val, x_var, y_var, z_var, 
                                           p_func, q_func, r_func, s_func, A, B, C, D, P_eye, Q_end, delta)
            
            # Check for degeneracy
            if is_degenerate(J):
                J = identity_matrix()
                
            # Newton step
            delta_sigma = solve_linear_system(J, F_val)
            sigma = sigma - delta_sigma
            
        else:
            # Max iterations reached, no solution for this system
            pass

    # 4. Determine if inside
    # If two valid t values exist, the segment between them is inside
    if len(valid_intersections) >= 2:
        t_min = min(valid_intersections)
        t_max = max(valid_intersections)
        # Calculate normal at intersection point for shading
        # ... (calculate normal using distance function gradient)
        return color_with_shading(t_min)
    else:
        return background_color

def map_sigma_to_params(sigma, fixed_param, fixed_val, x_var, y_var, z_var):
    u, v, w, t = 0.0, 0.0, 0.0, 0.0
    if fixed_param == 'u': u = fixed_val
    elif fixed_param == 'v': v = fixed_val
    elif fixed_param == 'w': w = fixed_val
    
    if x_var == 'u': u = sigma.x
    elif x_var == 'v': v = sigma.x
    elif x_var == 'w': w = sigma.x
    elif x_var == 't': t = sigma.x
    
    if y_var == 'u': u = sigma.y
    elif y_var == 'v': v = sigma.y
    elif y_var == 'w': w = sigma.y
    elif y_var == 't': t = sigma.y
    
    if z_var == 'u': u = sigma.z
    elif z_var == 'v': v = sigma.z
    elif z_var == 'w': w = sigma.z
    elif z_var == 't': t = sigma.z
    
    return u, v, w, t

def compute_jacobian_numerical(sigma, fixed_param, fixed_val, x_var, y_var, z_var, 
                               p_func, q_func, r_func, s_func, A, B, C, D, P_eye, Q_end, delta):
    # Helper to get F value for a given sigma
    def get_F(s):
        u, v, w, t = map_sigma_to_params(s, fixed_param, fixed_val, x_var, y_var, z_var)
        ray_point = P_eye + t * (Q_end - P_eye)
        p_val = p_func(u, v, w)
        q_val = q_func(u, v, w)
        r_val = r_func(u, v, w)
        s_val = s_func(u, v, w)
        solid_point = A * p_val + B * q_val + C * r_val + D * s_val
        return ray_point - solid_point

    J = mat3(0.0)
    # Column 1: dF/dx
    s_plus = sigma
    s_plus.x += delta
    s_minus = sigma
    s_minus.x -= delta
    J[0][0] = (get_F(s_plus).x - get_F(s_minus).x) / (2 * delta)
    J[1][0] = (get_F(s_plus).y - get_F(s_minus).y) / (2 * delta)
    J[2][0] = (get_F(s_plus).z - get_F(s_minus).z) / (2 * delta)
    
    # Column 2: dF/dy
    s_plus = sigma
    s_plus.y += delta
    s_minus = sigma
    s_minus.y -= delta
    J[0][1] = (get_F(s_plus).x - get_F(s_minus).x) / (2 * delta)
    J[1][1] = (get_F(s_plus).y - get_F(s_minus).y) / (2 * delta)
    J[2][1] = (get_F(s_plus).z - get_F(s_minus).z) / (2 * delta)
    
    # Column 3: dF/dz
    s_plus = sigma
    s_plus.z += delta
    s_minus = sigma
    s_minus.z -= delta
    J[0][2] = (get_F(s_plus).x - get_F(s_minus).x) / (2 * delta)
    J[1][2] = (get_F(s_plus).y - get_F(s_minus).y) / (2 * delta)
    J[2][2] = (get_F(s_plus).z - get_F(s_minus).z) / (2 * delta)
    
    return J

def is_degenerate(J):
    # Check determinant or condition number
    det = J[0][0] * (J[1][1]*J[2][2] - J[1][2]*J[2][1]) - \
          J[0][1] * (J[1][0]*J[2][2] - J[1][2]*J[2][0]) + \
          J[0][2] * (J[1][0]*J[2][1] - J[1][1]*J[2][0])
    return abs(det) < 1e-10
```

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel, but with a high constant factor due to solving 6 systems of equations, each requiring multiple Newton iterations (average 7-10 steps) [S1]. Each iteration involves evaluating the parametric functions and computing a 3x3 Jacobian via finite differences (6 function evaluations per Jacobian) [S1].
*   **Memory Complexity:** $O(1)$ per thread. Only vertex coordinates and a few temporary vectors are stored [S1].
*   **Measured Performance:** ~60 frames per second on general-purpose monitors using GPU acceleration (ShaderToy/GLSL) [S1].
*   **Strengths:** High image fidelity with complete absence of visual artifacts on faceted solids [S1]. Handles complex parametric definitions of solids [S1].
*   **Limitations:** Produces artifacts for curvilinear solids because Newton's method lacks sufficient accuracy for numerical solutions of non-linear systems in those cases [S1]. Degenerate Jacobians at the first iteration can cause loss of solution if not handled [S1]. Some edge case systems may have no solution or solutions outside $[0,1]$ [S1].
*   **When to use:** Rendering faceted solids (tetrahedrons, prisms) where artifact-free visualization is critical and the solid can be defined in point calculus [S1].
*   **When not to use:** Rendering curvilinear solids or when high performance is required for complex scenes with many solids, due to the iterative nature and lack of convergence guarantees for non-faceted shapes [S1].

## Background
The method was introduced in the paper "Visualization of Geometric Models of Faceted Solids in Point Calculus" by Konopatskiy, Ryabinin, and Bezdytniy [S1]. It builds on the concept of defining geometric solids as a three-parameter set of points in 3D space, originally formulated in earlier works [1, 2, 3] cited in [S1]. The approach uses point calculus mathematical apparatus to solve the intersection problem analytically/numerically, leveraging modern GPU hardware acceleration via ray marching [S1]. It was implemented in GLSL and demonstrated on ShaderToy [S1]. The method is part of the broader effort to integrate scientific visualization systems like SciVi [S1].

## Sources
[S1] Visualization of Geometric Models of Faceted Solids in Point Calculus, https://doi.org/10.26583/sv.15.1.05, fetched 2026-10-26
