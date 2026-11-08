# Bounded Forward Scattering Propagator (Dipole Model)

Also called: Boundary-Aware Propagator G_f^b, Boundary-aware propagator, Gbf, bounded propagator, final boundary-aware propagator

## Overview
The Bounded Forward Scattering Propagator ($G_f^b$) extends the Infinite Volume Forward Scattering Propagator [10181-infinite-volume-forward-scattering-propagator.md] to handle participating media within finite, bounded volumes by approximately satisfying absorbing boundary conditions. It employs a dipole method using a virtual source to enforce these boundaries, distinguishing itself from empirical corrections (such as those in [Frisvad et al. 2014]) by relying on the well-behaved nature of the underlying infinite volume propagator $G_f$ for regularization [S1].

## How it works
**Inputs:**
*   $\vec{x}_0, \hat{\omega}_0$: Outgoing position and direction.
*   $\vec{x}_1, \hat{\omega}_1$: Incoming position and direction.
*   $s$: Path length.
*   $\hat{n}_0, \hat{n}_1$: Surface normals at $\vec{x}_0$ and $\vec{x}_1$.
*   $\sigma_s', \sigma_a$: Reduced scattering coefficient and absorption coefficient.

**Outputs:**
*   $G_f^b(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_1, \hat{\omega}_1)$: The bounded propagator value.

**Data Structures:**
No specific complex data structures are required beyond standard vector and scalar types. The method relies on geometric calculations of normals and virtual source positions.

**Steps:**

1.  **Calculate Modified Normal ($\hat{n}_1^*$):**
    Determine the effective normal for the boundary plane at the incoming point $\vec{x}_1$.
    Let $\vec{R} = \vec{x}_0 - \vec{x}_1$.
    *   If $\vec{x}_0 = \vec{x}_1$, set $\hat{n}_1^* = \hat{n}_1$.
    *   Otherwise, calculate:
        $$ \hat{n}_1^* = \frac{\vec{R}}{|\vec{R}|} \times \frac{\hat{n}_1 \times \vec{R}}{|\hat{n}_1 \times \vec{R}|} $$
    *   **Pitfall:** If $\hat{n}_1$ is parallel to $\vec{R}$, the denominator $|\hat{n}_1 \times \vec{R}|$ is zero. In this case, set the propagator to 0 [S1].

2.  **Calculate Extrapolation Distance ($d_e$):**
    *   Compute reduced scattering albedo: $\alpha_0 = \sigma_s' / \sigma_t'$.
    *   Compute reduced transmission coefficient: $\sigma_t' = \sigma_s' + \sigma_a$.
    *   Compute diffusion constant: $D = 1/(3\sigma_t')$.
    *   Compute extrapolation distance: $d_e = 2.131 D / \sqrt{\alpha_0}$.

3.  **Define Virtual Source ($\vec{x}_v, \hat{\omega}_v$):**
    *   Set index-matching factor $A = 1$ (assuming no internal Fresnel reflection) [S1].
    *   Calculate virtual position: $\vec{x}_v = \vec{x}_1 + 2 A d_e \hat{n}_1^*$.
    *   Calculate virtual direction: $\hat{\omega}_v = \hat{\omega}_1^e - 2(\hat{\omega}_1^e \cdot \hat{n}_1^*)\hat{n}_1^*$, where $\hat{\omega}_1^e$ is the effective incoming direction (typically the normalized vector from $\vec{x}_1$ to $\vec{x}_0$, or the incident direction depending on the specific transport formulation; in the context of the propagator definition, it refers to the direction associated with the incoming point). *Note: The source defines $\hat{\omega}_v$ using $\hat{\omega}_i^e$. In the propagator formula, this corresponds to the direction at the incoming point.*

4.  **Evaluate Bounded Propagator:**
    The final value is determined by the relationship between the outgoing direction $\hat{\omega}_0$ and the modified normal $\hat{n}_1^*$:
    $$ G_f^b(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_1, \hat{\omega}_1) = \begin{cases} 0 & \text{if } \hat{\omega}_0 \cdot \hat{n}_1^* \geq 0, \\ G_f(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_1, \hat{\omega}_1) & \text{if } \hat{\omega}_0 \cdot \hat{n}_1^* < 0 \text{ and } \hat{\omega}_1^e \cdot \hat{n}_1^* \geq 0, \\ -G_f(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_v, \hat{\omega}_v) & \text{otherwise.} \end{cases} $$
    *   **Condition 1:** If the outgoing direction points into the effective half-space defined by $\hat{n}_1^*$ ($\hat{\omega}_0 \cdot \hat{n}_1^* \geq 0$), the propagator is 0. This handles the case where the "outgoing" query is actually pointing back into the medium relative to the boundary approximation [S1].
    *   **Condition 2:** If the outgoing direction points out of the medium ($\hat{\omega}_0 \cdot \hat{n}_1^* < 0$) but the incoming direction appears to point out of the medium ($\hat{\omega}_1^e \cdot \hat{n}_1^* \geq 0$), the propagator is 0. This is because the virtual source would not illuminate the medium in this planar approximation [S1].
    *   **Condition 3:** Otherwise, the propagator is the difference between the infinite volume propagator from the real source and the virtual source (dipole method): $G_f(\dots, \vec{x}_1, \dots) - G_f(\dots, \vec{x}_v, \dots)$. Note: The source formula presents this as a piecewise function where the third case is the negative of the virtual source term, implying the standard dipole subtraction $G_{real} - G_{virtual}$ when both conditions for non-zero output are met. The source explicitly writes:
        $$ G_f^b = \begin{cases} 0 & \text{if } \hat{\omega}_1 \cdot \hat{n}_1^* \geq 0 \quad (\text{Note: Source uses } \hat{\omega}_1 \text{ here, likely referring to outgoing direction at } \vec{x}_1 \text{ or a typo for } \hat{\omega}_0 \text{?}) \\ G_f(\dots, \vec{x}_1, \dots) & \text{otherwise...} \end{cases} $$
        *Correction based on Source Text:* The source text in Section 6.3 states:
        $$ G_f^b(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_1, \hat{\omega}_1) = \begin{cases} 0 & \text{if } \hat{\omega}_1 \cdot \hat{n}_1^* \geq 0, \\ G_f(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_1, \hat{\omega}_1) & \text{otherwise,} \\ -G_f(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_v, \hat{\omega}_v) & \end{cases} $$
        There is a notational ambiguity in the source's piecewise definition. The first condition checks $\hat{\omega}_1 \cdot \hat{n}_1^*$. In the context of "outgoing direction pointing into the medium", $\hat{\omega}_1$ usually denotes the direction *at* $\vec{x}_1$. However, standard transport notation often uses $\hat{\omega}_0$ for the outgoing direction of the *path segment* or the direction *leaving* the volume. The text says "outgoing direction to point into the effective half infinite medium". If $\vec{x}_1$ is the boundary point, the direction *leaving* the medium is $\hat{\omega}_1$ (if $\vec{x}_1$ is the exit point). Let us assume $\hat{\omega}_1$ in the source's formula refers to the direction at the incoming point $\vec{x}_1$ relative to the boundary normal.
        
        Re-reading carefully: "Note that it is also possible for the outgoing direction to point into the effective half infinite medium... we do not handle this separately." Then: "The final boundary-aware propagator then becomes... if $\hat{\omega}_1 \cdot \hat{n}_1^* \geq 0$ ... 0".
        It is highly likely that $\hat{\omega}_1$ in the source's formula represents the direction *at* $\vec{x}_1$ (the incoming point of the query, or outgoing from the previous segment). Given the symmetry of the problem, if $\vec{x}_1$ is the point on the boundary, $\hat{\omega}_1$ is the direction leaving $\vec{x}_1$. If this direction points *into* the medium ($\hat{\omega}_1 \cdot \hat{n}_1^* \geq 0$ assuming $\hat{n}_1^*$ points out), then no light escapes, so propagator is 0.
        
        The second case: "otherwise" (meaning $\hat{\omega}_1 \cdot \hat{n}_1^* < 0$, i.e., pointing out).
        The third case: "-G_f(..., x_v, ...)".
        
        This implies:
        If $\hat{\omega}_1 \cdot \hat{n}_1^* \geq 0$: 0.
        Else: $G_f(\dots, \vec{x}_1, \dots) - G_f(\dots, \vec{x}_v, \dots)$.
        
        However, the source also mentions: "incoming direction $\hat{\omega}_i^e$ can occasionally appear to point out of the medium... set the propagator to zero."
        This suggests a check on $\hat{\omega}_1^e \cdot \hat{n}_1^*$.
        
        Let's synthesize the complete logic from the text segments:
        1. Check if $\hat{\omega}_1 \cdot \hat{n}_1^* \geq 0$. If so, return 0. (Outgoing direction points into medium).
        2. Check if $\hat{\omega}_1^e \cdot \hat{n}_1^* \geq 0$. If so, return 0. (Incoming direction points out of medium, so virtual source doesn't illuminate).
        3. Otherwise, return $G_f(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_1, \hat{\omega}_1) - G_f(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_v, \hat{\omega}_v)$.

    *   **Pitfall:** If $\hat{\omega}_1^e \cdot \hat{n}_1^* \geq 0$, the source would not illuminate the medium in the planar approximation. Set propagator to 0 [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bounded_forward_scattering_propagator(s, x0, w0, x1, w1, n1, sigma_s_prime, sigma_a):
    # 1. Calculate Modified Normal n1_star
    R = x0 - x1
    if norm(R) < 1e-9:
        n1_star = n1
    else:
        R_norm = R / norm(R)
        cross_prod = cross(n1, R)
        cross_norm = norm(cross_prod)
        if cross_norm < 1e-9:
            # Ill-defined normal: n1 parallel to R
            return 0.0
        n1_star = cross(R_norm, cross_prod / cross_norm)

    # 2. Calculate Extrapolation Distance de
    sigma_t_prime = sigma_s_prime + sigma_a
    if sigma_t_prime == 0:
        return 0.0 # Avoid division by zero
    alpha_0 = sigma_s_prime / sigma_t_prime
    D = 1.0 / (3.0 * sigma_t_prime)
    if alpha_0 <= 0:
        return 0.0 # Avoid division by zero or sqrt of negative
    d_e = 2.131 * D / sqrt(alpha_0)

    # 3. Define Virtual Source
    A = 1.0 # Index-matched boundary
    x_v = x1 + 2.0 * A * d_e * n1_star
    
    # Effective incoming direction at x1. 
    # In the context of the propagator G(x0, w0 -> x1, w1), 
    # w1 is the direction arriving at x1. 
    # The source uses w_i^e. We assume w1 is the direction vector.
    # Reflection of w1 across n1_star
    w_v = w1 - 2.0 * dot(w1, n1_star) * n1_star

    # 4. Evaluate Propagator
    # Check outgoing direction relative to boundary
    if dot(w1, n1_star) >= 0:
        return 0.0
    
    # Check incoming direction relative to boundary
    if dot(w1, n1_star) >= 0: # Redundant with above? 
        # Source says: "incoming direction w_i^e can occasionally appear to point out... n_i^* . w_i^e > 0"
        # If w1 is the incoming direction, this check is the same as the first one if w1 is the direction.
        # However, w1 is the direction *at* x1. 
        # Let's assume the source distinguishes between the geometric direction and the transport direction.
        # For implementation, if w1 points into the medium (dot >= 0), it's 0.
        pass

    # The source formula:
    # if w1 . n1* >= 0: 0
    # else: G_f(...) - G_f(...)
    
    # But it also says: "incoming direction ... point out ... set to 0".
    # If w1 is the incoming direction, dot(w1, n1*) > 0 means it points OUT (assuming n1* points OUT).
    # Wait, standard convention: n points OUT.
    # If w points OUT, dot(w, n) > 0.
    # If w points IN, dot(w, n) < 0.
    
    # Source: "outgoing direction to point into the effective half infinite medium... n1* . w1 >= 0 ... 0"
    # This implies n1* points IN? Or w1 is defined differently?
    # "n1* . w1 >= 0" -> 0.
    # If n1* points OUT, and w1 points OUT, dot > 0. Light escapes. Propagator should be non-zero?
    # No, the propagator G(x0, w0 -> x1, w1) describes transport FROM x0 TO x1.
    # If x1 is on the boundary, and we are evaluating the contribution TO x1.
    
    # Let's stick strictly to the source's conditional logic:
    # Condition 1: if dot(w1, n1_star) >= 0: return 0.
    
    # Condition 2: The source mentions "incoming direction ... point out ... set to 0".
    # If w1 is the incoming direction, and it points OUT (dot > 0), then 0.
    # This is the same as Condition 1.
    
    # Therefore, the logic is:
    if dot(w1, n1_star) >= 0:
        return 0.0
    
    # Otherwise, calculate G_f
    g_real = infinite_volume_forward_scattering_propagator(s, x0, w0, x1, w1)
    g_virtual = infinite_volume_forward_scattering_propagator(s, x0, w0, x_v, w_v)
    
    return g_real - g_virtual
```

## Subtopics

### Reciprocal Bounded Propagator
A symmetrical combination of the bounded propagator and its reverse to enforce reciprocity, which is broken by the virtual source dependency on the incoming normal. The formula is $\frac{1}{2} [ G_f^b(s, \vec{x}_0, \hat{\omega}_0, \vec{x}_i, \hat{\omega}_i) + G_f^b(s, \vec{x}_i, -\hat{\omega}_i, \vec{x}_0, -\hat{\omega}_0) ]$. Reciprocity holds only if $\hat{n}_i = \hat{n}_o$ (planar boundaries) or if $\hat{n}_i, \hat{n}_o$ and $\vec{x}_o - \vec{x}_i$ are coplanar. In practice, the simple $G_f^b$ is used because the difference is negligible and unidirectional path tracing does not strictly require reciprocity [S1].

## Operating characteristics
*   **Complexity:** The method requires evaluating the infinite volume propagator $G_f$ twice (once for the real source, once for the virtual source) plus geometric calculations for the modified normal and virtual source. The complexity is dominated by $G_f$.
*   **Performance:** Not given in the sources.
*   **Strengths:** Handles bounded volumes with absorbing boundary conditions without empirical distance corrections, leveraging the well-behaved nature of $G_f$.
*   **Limitations:** Breaks reciprocity unless the symmetrical combination is used. Ill-defined normals occur when the surface normal is parallel to the displacement vector.
*   **When to use:** When rendering bounded participating media with forward scattering dipole models and index-matched boundaries.

## Background
The method is based on the dipole model for diffusion approximation, originally studied in physics [Bell and Glasstone 1970; Lamarsh 1975] and computer graphics [d'Eon 2014; Habel et al. 2013; Jensen et al. 2001]. It was introduced in the context of a forward scattering dipole model derived from a functional integral approximation [S1]. It differs from Frisvad et al. [2014] by not requiring an empirical distance correction to offset the real source, as $G_f$ is well-behaved except for the designed delta distribution at short lengths.

## Sources
[S1] A forward scattering dipole model from a functional integral approximation, https://doi.org/10.1145/3072959.3073681, fetched 2026-10-26
