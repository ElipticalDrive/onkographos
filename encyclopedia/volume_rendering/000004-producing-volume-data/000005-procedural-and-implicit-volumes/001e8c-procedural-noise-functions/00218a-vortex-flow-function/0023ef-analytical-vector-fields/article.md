# Analytical Vector Fields

Also called: Analytic Vector Fields

## Overview
Analytical vector fields generate flow data by evaluating explicit mathematical functions of position and time, rather than interpolating from simulation grids or particle systems [S1, unofficial]. This approach allows for arbitrary resolution and seamless infinite domains, making it suitable for procedural volume rendering where memory constraints prohibit storing large velocity fields [S1, unofficial]. The strategy produces visually appealing results but is not applicable to actual simulation data [S1, unofficial].

## Shared principles
All analytical vector fields share the property that the velocity $\mathbf{u}$ at any point $\mathbf{x}$ and time $t$ is computed via a closed-form function $\mathbf{u} = \mathbf{F}(\mathbf{x}, t)$ [S1, unofficial]. To advect particles or trace rays through the field, the position update follows the ordinary differential equation $\frac{d\mathbf{x}}{dt} = \mathbf{F}(\mathbf{x}, t)$. Standard implementations use a fourth-order Runge-Kutta (RK4) integrator to compute the next position $\mathbf{x}_{i+1}$ from $\mathbf{x}_i$ with step size $\delta$ [S1, unofficial]:
$$ \mathbf{x}_{i+1} = \mathbf{x}_i + \frac{\delta}{6} (\mathbf{v}_0 + 2\mathbf{v}_1 + 2\mathbf{v}_2 + \mathbf{v}_3) $$
where the intermediate velocities $\mathbf{v}_k$ are samples of $\mathbf{F}$ at predicted positions within the step [S1, unofficial]. For real-time rendering, the function time $t_{func}$ and integration step size $\delta$ must be synchronized to ensure temporal consistency of the flow [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rk4_step(x, t, dt, field_func):
    """Perform one RK4 integration step for an analytical vector field."""
    v0 = field_func(x, t)
    v1 = field_func(x + 0.5 * dt * v0, t + 0.5 * dt)
    v2 = field_func(x + 0.5 * dt * v1, t + 0.5 * dt)
    v3 = field_func(x + dt * v2, t + dt)
    return x + (dt / 6.0) * (v0 + 2.0 * v1 + 2.0 * v2 + v3)
```

## Subtopics

### [Gaussian Normal Field Definition](9614-gaussian-normal-field-definition.md)
Derives an explicit surface normal field from a Gaussian Splatting representation by computing the gradient of the log-vacancy. It differs from procedural alternatives like curl noise by providing a deterministic, closed-form normal estimate rather than stochastic patterns. Choose this method when supervising training to enforce proper surface wrapping or when extracting meshes from splats.

### [Curl Noise Generation](10372-curl-noise-generation.md)
Produces a divergence-free vector field by computing the curl of a scalar noise field, ensuring the flow has no sources or sinks. It is distinct from simulated fluid dynamics because it relies on noise derivatives rather than solving Navier-Stokes equations. Choose this method for generating smooth, organic flow patterns for particle advection or procedural texturing.
- Linear Field: A constant vector field that returns the same direction for all inputs, defined as $\mathbf{u}(\mathbf{v}) = (1, 1, 1)^T$. It is used to visualize basic volume rendering aspects because it introduces no deformation or shear if the seeding surface is aligned with the flow.
- ABC-Flow: The Arnold-Beltrami-Childress (ABC) flow is a divergence-free analytical turbulence generator defined by $\mathbf{u}(\mathbf{v}) = \begin{pmatrix} A \sin(v_z) + C \cos(v_y) \\ B \sin(v_x) + A \cos(v_z) \\ C \sin(v_y) + B \cos(v_x) \end{pmatrix}$ where $A, B, C$ are design parameters. It is commonly used to evaluate new rendering approaches which require vector field data.
- N-Force Field: An analytical field composed of $N_{core}$ rotating cores used to simulate swirling motion. The field is sampled by projecting the position $\mathbf{v}$ onto the XZ plane and calculating the contribution of each core $i$:
  $$ f_i = \text{rot}_Y(\pm 90^\circ) \left[ \begin{pmatrix} \cos(\gamma_i) \\ 0 \\ \sin(\gamma_i) \end{pmatrix} - \begin{pmatrix} v_x \\ 0 \\ v_z \end{pmatrix} \right] $$
  The output is the average of these contributions:
  $$ \mathbf{v}_{\text{out}} = \frac{1}{N_{\text{core}}} \sum_{i=0}^{N_{\text{core}}-1} \left[ f_i + \begin{pmatrix} 0 \\ e^{-|f_i|^2} \\ 0 \end{pmatrix} \right] $$
  The sign of the 90° rotation determines the spin direction relative to the global orientation $\gamma_i$.
- Spherical Attractor: A primitive function that creates paths radially towards or away from a center point, used to simulate effects like breezes or wakes.
  **Step:**
  1. Calculate distance from the attractor center: $\text{dist} = \sqrt{\text{DOT}(\mathbf{point} - \mathbf{FF.CENTER}, \mathbf{point} - \mathbf{FF.CENTER})}$.
  2. Set `density_scaling` to 1.0 and `velocity` to 1.0.
  3. Calculate `percent_to_use` based on distance:
     * If $\text{dist} > \text{FF.DISTANCE}$, `percent_to_use` = 0.
     * If $\text{dist} < \text{FF.FALLOFF\_START}$, `percent_to_use` = 1.0.
     * Otherwise, interpolate using cosine falloff:
       $$ \text{percent\_to\_use} = \frac{\cos(d_2 \pi) + 1.0}{2.0} $$
       where $d_2 = \frac{\text{dist} - \text{FF.FALLOFF\_START}}{\text{FF.DISTANCE} - \text{FF.FALLOFF\_START}}$.
  **Parameters:**
  * $\text{FF.CENTER}$: The center of the attractor.
  * $\text{FF.DISTANCE}$: The effective distance of attraction.
  * $\text{FF.FALLOFF\_START}$: The distance where falloff from the attractor path to the default path begins.

## Background
Analytical vector fields were introduced as a method to test smoke volume renderers by providing deterministic, mathematically defined flow fields without the overhead of simulation data [S1, unofficial]. The approach allows for real-time behavior by avoiding grid lookups, relying instead on direct function evaluation [S1, unofficial]. Variations such as spherical attractors were developed to create specific visual effects like breezes and wakes by animating the location and strength of attraction points [S2, unofficial].

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S2] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
