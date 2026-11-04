# ERP/Baumgarte Collision Resolution
Also called: SDF Physics Collision, Terrain Collision Resolve
## Overview
ERP/Baumgarte Collision Resolution is a constraint-based method for resolving object penetration in terrain physics using Signed Distance Fields (SDF). It corrects positional drift via Baumgarte stabilization and eliminates inward velocity components to prevent tunneling. It distinguishes itself from simple positional correction by combining velocity damping with a proportional-derivative-like impulse application, ensuring stable contact without excessive jitter [S1, unofficial].

## How it works
**Inputs:**
- `pos`: Current object position (vector).
- `oldPos`: Previous frame object position (vector).
- `vel`: Current object velocity (vector).
- `mass`: Object mass (scalar).
- `contactOffset`: SDF distance threshold for collision detection (scalar).
- `slop`: Allowed penetration depth before resolution triggers (scalar).
- `beta`: Baumgarte stabilization factor (scalar, typically $0.2 \le \beta \le 0.8$).
- `frictionMu`: Coefficient of friction (scalar).
- `dt`: Time step duration (scalar).

**Outputs:**
- `newVel`: Updated velocity after collision resolution (vector).

**Data Structures:**
No complex data structures are required. The method operates on scalar and vector primitives. The SDF is queried via a function `SDF(pos)` returning distance, and `EstimateGradient(pos)` returning the surface normal gradient.

**Steps:**

1.  **Early Exit:** If `dt <= 0` or the magnitude of the SDF normal is near zero ($< 10^{-10}$), return `vel` unchanged [S1, unofficial].
2.  **Surface Normal:** Compute the surface normal $\mathbf{n}$ by normalizing the gradient of the SDF at the old position:
    $$ \mathbf{n} = \frac{\nabla \text{SDF}(\text{oldPos})}{\|\nabla \text{SDF}(\text{oldPos})\|} $$
3.  **Penetration Depth:** Calculate penetration $p$. Let $d_{new} = \text{SDF}(\text{pos})$ and $d_{old} = \text{SDF}(\text{oldPos})$.
    $$ p = \max(0, (\text{contactOffset} - d_{new}) - \text{slop}) $$
    If $p \le 0$, return `vel` unchanged [S1, unofficial].
4.  **Collision Detection Check:** A collision is considered to have occurred if the object crossed the surface boundary:
    $$ (d_{old} > 0 \land d_{new} < 0) \lor (d_{old} < 0 \land d_{new} > 0) $$
    If no collision occurred, return `vel` unchanged [S1, unofficial].
5.  **Velocity Bias (Baumgarte Stabilization):** Calculate the incoming normal velocity $v_N$:
    $$ v_N = \mathbf{v} \cdot \mathbf{n} $$
    Calculate the velocity bias $\Delta v_N$:
    $$ \Delta v_N = \begin{cases} -v_N + \beta \frac{p}{\max(10^{-6}, dt)} & \text{if } v_N < 0 \\ 0 & \text{otherwise} \end{cases} $$
    Update velocity: $\mathbf{v} \leftarrow \mathbf{v} + \Delta v_N \mathbf{n}$ [S1, unofficial].
6.  **Inward Velocity Guard:** Ensure no inward velocity remains. If $\mathbf{v} \cdot \mathbf{n} < 0$, subtract the inward component:
    $$ \mathbf{v} \leftarrow \mathbf{v} - (\mathbf{v} \cdot \mathbf{n}) \mathbf{n} $$
7.  **Tangential Friction:** Calculate the normal impulse magnitude $J_n$:
    $$ J_n = \text{mass} \cdot \max(0, \Delta v_N) $$
    If $\text{frictionMu} > 0$ and $J_n > 0$:
    - Calculate tangential velocity $\mathbf{v}_T = \mathbf{v} - (\mathbf{v} \cdot \mathbf{n}) \mathbf{n}$.
    - If $\|\mathbf{v}_T\| > 10^{-6}$:
        - Calculate max friction impulse $J_{t,\max} = \text{frictionMu} \cdot J_n$.
        - Calculate needed impulse $J_{t,\text{needed}} = \text{mass} \cdot \|\mathbf{v}_T\|$.
        - Apply friction impulse $\mathbf{J}_t = -\frac{\mathbf{v}_T}{\|\mathbf{v}_T\|} \min(J_{t,\text{needed}}, J_{t,\max})$.
        - Update velocity: $\mathbf{v} \leftarrow \mathbf{v} + \frac{\mathbf{J}_t}{\text{mass}}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resolve_collision(pos, old_pos, vel, mass, contact_offset, slop, beta, friction_mu, dt):
    if dt <= 0:
        return vel
    
    # Estimate gradient at old position
    grad = estimate_gradient(old_pos)
    grad_mag = magnitude(grad)
    
    if grad_mag < 1e-10:
        return vel
    
    # Normalize normal
    n = normalize(grad)
    
    # Get SDF distances
    new_dist = sdf_distance(pos)
    old_dist = sdf_distance(old_pos)
    
    # Check for collision crossing
    if not ((old_dist > 0 and new_dist < 0) or (old_dist < 0 and new_dist > 0)):
        return vel
    
    # Calculate penetration
    penetration = max(0.0, (contact_offset - new_dist) - slop)
    
    if penetration <= 0:
        return vel
    
    # Calculate incoming normal velocity
    v_n = dot(vel, n)
    
    # Calculate velocity bias
    dv_n = 0.0
    if v_n < 0:
        dv_n = -v_n + beta * (penetration / max(1e-6, dt))
    
    # Apply normal impulse
    vel = vel + dv_n * n
    
    # Guard against remaining inward velocity
    if dot(vel, n) < 0:
        vel = vel - (dot(vel, n) * n)
    
    # Tangential friction
    j_n = mass * max(0.0, dv_n)
    
    if friction_mu > 0 and j_n > 0:
        # Tangential velocity
        v_t = vel - (dot(vel, n) * n)
        v_t_mag = magnitude(v_t)
        
        if v_t_mag > 1e-6:
            j_t_max = friction_mu * j_n
            j_t_needed = mass * v_t_mag
            j_t_magnitude = min(j_t_needed, j_t_max)
            
            # Apply friction
            vel = vel - (normalize(v_t) * j_t_magnitude / mass)
            
    return vel
```

**Parameters:**
- `beta`: Typically between $0.2$ and $0.8$. Higher values reduce penetration but may cause instability [S1, unofficial].
- `contactOffset`: Defines the "soft" collision boundary. Not given in the sources.
- `slop`: Small tolerance to prevent jitter. Not given in the sources.

**Edge Cases:**
- Zero time step `dt`: Handled by early exit [S1, unofficial].
- Zero gradient magnitude: Handled by early exit [S1, unofficial].
- Zero penetration: Handled by early exit [S1, unofficial].
- Division by zero in Baumgarte term: Guarded by `max(1e-6, dt)` [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per collision resolution step, assuming $O(1)$ SDF evaluation and gradient estimation.
- **Memory Complexity:** $O(1)$ auxiliary memory.
- **Strengths:** Stable resolution of penetration without requiring iterative constraint solvers. Combines positional correction (via Baumgarte) with velocity damping.
- **Limitations:** Accuracy depends on the quality of the SDF gradient. May exhibit jitter if `beta` is too high or `slop` is too low. Not suitable for complex multi-body contacts without additional broad-phase or iterative resolution.
- **When to use:** Single-object terrain collision in real-time simulations where performance is critical and SDFs are available.
- **When not to use:** Complex multi-body collisions, non-SDF based collision detection, or scenarios requiring precise static equilibrium without iterative solvers.

## Background
The method is derived from physics engine implementations using SDFs for terrain collision, specifically referencing code from the `Marching-Cubes-Terrain` repository [S1, unofficial]. The term "Baumgarte stabilization" refers to a technique used in constraint solving to reduce positional drift by feeding back position errors into velocity corrections, originally popularized in rigid body dynamics [S1, unofficial]. The specific implementation combines this with ERP (Error Reduction Parameter) concepts, though the exact ERP formulation is not detailed in the source, relying instead on the `beta` parameter for stabilization [S1, unofficial].

## Sources
[S1] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
