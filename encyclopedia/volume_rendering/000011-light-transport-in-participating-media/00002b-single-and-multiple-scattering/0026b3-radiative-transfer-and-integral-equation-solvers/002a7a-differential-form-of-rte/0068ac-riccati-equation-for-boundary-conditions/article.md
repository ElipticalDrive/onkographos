# Riccati Equation for Boundary Conditions

Also called: BRDF Matrix Differential Equation

## Overview
A method to solve the two-point boundary value problem for radiance transport in layered participating media (specifically vegetation) by converting it into an initial value problem using a BRDF matrix $F(z)$. It sets itself apart from iterative shooting or relaxation methods by deriving a non-linear ordinary differential equation (Riccati equation) for the reflection matrix, allowing incremental integration [S1, book].

## How it works
**Inputs and Outputs:**
- **Input:** Ground BRDF matrix $F(0)$, scattering matrices $P_{uu}, P_{dd}, P_{ud}, P_{du}$, and the domain height $h$.
- **Output:** Upward radiance vector $I_u(z)$ and downward radiance vector $I_d(z)$ for all $z \in [0, h]$.

**Data Structures:**
- **Radiance Vectors:** $I_u(z)$ and $I_d(z)$ are row vectors of size $N$, where $N$ is the number of direction bins in a hemisphere. $N = (m-1)n + 1$, with $m$ latitude bins and $n$ longitude bins [S1, book].
- **Scattering Matrices:** $P_{uu}, P_{dd}, P_{ud}, P_{du}$ are $N \times N$ matrices representing scattering between upward and downward directions [S1, book].
- **BRDF Matrix:** $F(z)$ is an $N \times N$ matrix representing the bidirectional reflectance distribution function for the layer $[0, z]$, including multiple scattering [S1, book].

**Steps:**
1. **Initialize:** Set the initial condition for the Riccati equation at the ground level: $F(0) = F_{ground}$, where $F_{ground}$ is the known BRDF matrix of the ground [S1, book].
2. **Integrate Riccati Equation:** Solve the differential equation for $F(z)$ incrementally upwards from $z=0$ to $z=h$:
   $$ \frac{dF(z)}{dz} = P_{du} + F(z)P_{dd} + P_{ud}F(z) + F(z)P_{dd}F(z) $$
   This is a non-linear matrix Riccati equation [S1, book].
3. **Compute Boundary Condition:** Once $F(h)$ is known, compute the upward radiance at the top of the canopy:
   $$ I_u(h) = I_d(h) F(h) $$
   Here, $I_d(h)$ is the downward radiance at the top (typically solar illumination) [S1, book].
4. **Integrate Radiance Transport:** Use $I_u(h)$ as the initial condition and integrate the radiance transport equations (equation 3 in source) incrementally downwards from $z=h$ to $z=0$ to find $I_u(z)$ and $I_d(z)$ throughout the domain [S1, book].

**Parameters:**
- **Symmetry Assumption:** Scattering terms depend directionally only on $\theta, \theta'$, and $|\phi - \phi'|$. This allows the system to split into independent groups [S1, book].
- **Simplification:** $P_{uu} = P_{dd}$ and $P_{ud} = P_{du}$ is assumed if leaves are two-sided translucent and stems/branches are cylindrical [S1, book].

**Edge Cases and Pitfalls:**
- **Precision:** Matrix multiplications are $O(N^3)$, which becomes slow for fine angular subdivisions [S1, book].
- **Boundary Conditions:** The method requires a known ground BRDF $F(0)$ [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solve_radiance_transport(h, F_ground, P_uu, P_dd, P_ud, P_du, I_d_h):
    """
    Solves radiance transport using Riccati equation for boundary conditions.
    
    Args:
        h: Height of the domain.
        F_ground: N x N matrix, BRDF of the ground at z=0.
        P_uu, P_dd, P_ud, P_du: N x N scattering matrices.
        I_d_h: Row vector, downward radiance at z=h.
        
    Returns:
        I_u: Array of shape (h_steps, N), upward radiance at each z.
        I_d: Array of shape (h_steps, N), downward radiance at each z.
    """
    N = F_ground.shape[0]
    
    # Step 1: Integrate Riccati Equation from z=0 to z=h
    # F(z) is the BRDF matrix for the layer [0, z]
    # dF/dz = P_du + F*P_dd + P_ud*F + F*P_dd*F
    
    F = F_ground.copy()
    z = 0.0
    dz = h / 100.0 # Example step size; use adaptive Runge-Kutta in practice
    
    while z < h:
        # Compute derivative of F
        dF = P_du + F @ P_dd + P_ud @ F + F @ P_dd @ F
        
        # Update F using RK4 or similar adaptive integrator
        # F_new = F + dF * dz (Euler step for illustration)
        F = F + dF * dz
        z += dz
        
    F_h = F # F(h)
    
    # Step 2: Compute I_u(h)
    I_u_h = I_d_h @ F_h
    
    # Step 3: Integrate Radiance Transport downwards from z=h to z=0
    # Equations (3) in source are not explicitly given in the text,
    # but generally involve dI/dz = -sigma_t * I + scattering terms.
    # We assume a generic integration function for the radiance transport ODE system.
    
    I_u = []
    I_d = []
    z = h
    I_u_curr = I_u_h
    I_d_curr = I_d_h
    
    while z >= 0:
        I_u.append(I_u_curr)
        I_d.append(I_d_curr)
        
        # Compute derivatives for radiance transport
        # dI_u/dz and dI_d/dz depend on P matrices and current I_u, I_d
        # dI_u/dz = -P_uu * I_u - P_ud * I_d (Example structure, source eq 3 not fully detailed)
        # dI_d/dz = P_du * I_u + P_dd * I_d
        
        # Placeholder for actual radiance transport ODE integration
        # In practice, use the same adaptive Runge-Kutta as for F(z)
        dI_u = -P_uu @ I_u_curr - P_ud @ I_d_curr
        dI_d = P_du @ I_u_curr + P_dd @ I_d_curr
        
        I_u_curr = I_u_curr + dI_u * (-dz) # Moving downwards
        I_d_curr = I_d_curr + dI_d * (-dz)
        z -= dz
        
    return I_u, I_d
```

## Subtopics

### [Plane Parallel Radiance Transport for Vegetation](20250-plane-parallel-radiance-transport-for-vegetation.md)
A method to compute global illumination in dense vegetation by assuming radiance depends positionally only on height $z$ and angularly on direction. It reduces PDEs to ODEs in $z$, solved via Runge-Kutta. It builds on the Riccati Equation for Boundary Conditions to handle the two-point boundary value problem.

### Fourier Series Expansion for Symmetry
A technique to reduce computational cost by expanding radiance vectors and matrices into a finite Fourier series in azimuth angle $\phi$, exploiting rotational symmetry. It splits the system of $2N$ coupled differential equations into $n+1$ independent groups of $2m$ equations, reducing time complexity from $O(N^3)$ to $O(nm^3)$ per step [S1, book].

## Operating characteristics
- **Time Complexity:** $O(N^3)$ per integration step for the Riccati equation, where $N$ is the number of direction bins in a hemisphere ($N = (m-1)n + 1$) [S1, book].
- **Memory Complexity:** $O(N^2)$ to store the $N \times N$ matrices $F(z)$ and $P$ matrices [S1, book].
- **Scaling:** Performance degrades significantly with fine angular subdivisions due to the cubic complexity of matrix multiplications [S1, book].
- **Strengths:** Avoids complicated iterations required by shooting and relaxation methods for two-point boundary value problems [S1, book].
- **Limitations:** Computationally expensive for high-resolution angular discretization [S1, book].
- **When to Use:** When solving radiance transport in layered media with known ground BRDF and where iterative methods are too slow or unstable [S1, book].
- **When Not to Use:** When angular resolution is very high and computational resources are limited [S1, book].

## Background
Originated in the context of global illumination in vegetation, specifically for plane-parallel radiance transport. The method was developed to address the computational inefficiency of shooting and relaxation methods for two-point boundary value problems in radiative transfer [S1, book]. It uses adaptive Runge-Kutta routines from the Numerical Recipes package for integration [S1, book].

## Sources
[S1] Plane parallel radiance transport for global illumination in vegetation, https://www.osti.gov/biblio/649229, fetched 2026-10-26 (book)
