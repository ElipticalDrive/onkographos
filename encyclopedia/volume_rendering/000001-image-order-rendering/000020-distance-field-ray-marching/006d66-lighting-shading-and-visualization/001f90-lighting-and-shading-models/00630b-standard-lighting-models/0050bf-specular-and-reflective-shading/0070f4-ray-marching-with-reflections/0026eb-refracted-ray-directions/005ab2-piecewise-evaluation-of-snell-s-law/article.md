# Piecewise evaluation of Snell's law

Also called: Piecewise Snell's law, Discrete refraction approximation

## Overview
An approximation technique for continuous refraction that repeatedly evaluates Snell's law at discrete steps along a ray path [S1]. It serves as a middle ground between constant refraction (single-step) and higher-order numerical integration (e.g., RK4) for rays traversing media with spatially varying refractive indices [S1]. Unlike [Refracted Ray Directions](3915-refracted-ray-directions.md), which handles discrete interfaces, this method approximates the curved trajectory within a gradient medium by treating it as a sequence of small, homogeneous slabs [S1].

## How it works
**Inputs:**
- Ray origin $O$ and direction $D$.
- Refractive index field $n(\mathbf{x})$, typically defined as a continuous function or sampled on a grid.
- Step size $h$ (distance between evaluation points).

**Outputs:**
- Ray trajectory points and final exit direction.

**Data Structures:**
- Current position $\mathbf{x}_i$ and direction $\mathbf{d}_i$.
- Refractive index $n_i = n(\mathbf{x}_i)$ and gradient $\nabla n_i = \nabla n(\mathbf{x}_i)$.

**Steps:**
1. Initialize $\mathbf{x}_0 = O$, $\mathbf{d}_0 = D$. Calculate $n_0 = n(\mathbf{x}_0)$.
2. For each step $i = 0, 1, \dots$:
   a. Compute the refractive index gradient $\nabla n_i$ at $\mathbf{x}_i$. If the field is discrete, use finite differences [G].
   b. Determine the normal to the isophote (surface of constant index) at $\mathbf{x}_i$: $\mathbf{N}_i = \frac{\nabla n_i}{\|\nabla n_i\|}$.
   c. Calculate the angle of incidence $\theta_i$ between $-\mathbf{d}_i$ and $\mathbf{N}_i$: $\cos \theta_i = -\mathbf{d}_i \cdot \mathbf{N}_i$.
   d. Calculate the refractive index at the next step $\mathbf{x}_{i+1} = \mathbf{x}_i + h \mathbf{d}_i$: $n_{i+1} = n(\mathbf{x}_{i+1})$.
   e. Apply Snell's law to find the new direction $\mathbf{d}_{i+1}$. The change in direction $\Delta \mathbf{d}$ is proportional to the gradient component perpendicular to the ray. A common implementation uses the vector form of Snell's law for small steps:
      $$ \mathbf{d}_{i+1} = \frac{n_i}{n_{i+1}} \left( \mathbf{d}_i - (\mathbf{d}_i \cdot \mathbf{N}_i)\mathbf{N}_i \right) + \sqrt{1 - \left(\frac{n_i}{n_{i+1}}\right)^2 \left(1 - (\mathbf{d}_i \cdot \mathbf{N}_i)^2\right)} \mathbf{N}_i $$
      *Note: This formula assumes the ray bends towards the higher index. If $n_{i+1} < n_i$, total internal reflection may occur if the argument of the square root is negative [G].*
   f. Update $\mathbf{x}_i \leftarrow \mathbf{x}_{i+1}$, $\mathbf{d}_i \leftarrow \mathbf{d}_{i+1}$.
   g. Check termination condition (exit volume or max steps).

**Parameters:**
- Step size $h$: Not given in the sources. Smaller $h$ increases accuracy but cost.

**Edge cases and pitfalls:**
- Fails to sufficiently reproduce the shape of caustics and introduces significant discrepancies compared to analytical or RK4 solutions [S1].
- Total internal reflection must be handled if the ray attempts to exit a high-index region into a lower-index one at a steep angle [G].
- Precision errors accumulate over many steps [G].

## Subtopics

### GRIN Modeling
Modeling the porcine eye lens as a Gradient Refractive Index (GRIN) medium by dividing it into two regions (anterior and middle) and converting a polar-coordinate profile to a Cartesian polynomial for Zemax 'Gradient 3' surfaces [S2]. It sets itself apart by providing a specific parameterization for biological tissue simulation rather than a general ray-marching algorithm [S2]. Choose it when simulating light propagation through the porcine eye lens for OCT applications [S2]. Link: [Thick Lens (Snell's Law Refraction)](21374-thick-lens-snell-s-law-refraction.md) (related to lens modeling).

**Definition:**
The GRIN profile is defined in polar coordinates as:
$$ n(\rho,\theta) = n_N - (n_N - n_S) \cdot \left(\frac{\rho}{\rho_S}\right)^{p(\theta)} $$
where $\rho_S$ is the distance from lens center to surface, $\rho$ is distance from center to point, $\theta$ is angle from optical axis, and $p(\theta)$ is an exponent function [S2].

**Formula:**
For Zemax compatibility, the profile is converted to a Cartesian polynomial $n(z)$ along the optical axis ($\theta=0$):
$$ n(z) = n_0 + N_{z1} \cdot z + N_{z2} \cdot z^2 $$
where $z$ is distance from surface vertex, and $N_{z1}, N_{z2}$ are solved by matching $n(z)$ at midpoint and endpoint of lens thickness [S2].

**Parameters:**
- $n_N = 1.443$ (center index), $n_S = 1.362$ (surface index) [S2].
- Exponent axial value $p_1 = 2.62$; Exponent meridional range $p_2 = 3.56$ to $5.18$ [S2].
- Simplified model uses $p(0) = p_1 = 2.62$ [S2].

**Layout:**
Table 2: GRIN Parameters with fields: Surface, $n_0$, $N_{z1}$, $N_{z2}$ [S2].
- Anterior Lens: $n_0=1.362$, $N_{z1}=0.063$, $N_{z2}=-0.012$ [S2].
- Middle Lens: $n_0=1.443$, $N_{z1}=6.481 \times 10^{-3}$, $N_{z2}=-5.734 \times 10^{-3}$ [S2].

## Operating characteristics
- **Time Complexity:** Linear in the number of steps $N$. Each step requires evaluating $n(\mathbf{x})$ and $\nabla n(\mathbf{x})$.
- **Memory Complexity:** Constant per step, plus storage for the refractive index field.
- **Measured Performance:** The piecewise approximation shows significant discrepancies compared to analytical or RK4 solutions, particularly in reproducing caustic shapes [S1].
- **Strengths:** Simpler to implement than RK4; more accurate than constant refraction for smooth gradients.
- **Limitations:** Fails to accurately reproduce caustics; accuracy depends heavily on step size [S1].
- **When to use:** When a balance between speed and accuracy is needed, and caustic fidelity is not critical.
- **When not to use:** When high-fidelity caustics are required (use RK4 or analytical solutions) [S1].

## Background
Originated in the context of refractive radiative transfer equation simulations [S1]. It was compared against constant refraction and RK4 methods in studies involving photon trajectories through gradient media [S1]. The GRIN modeling variant was developed for simulating porcine eye lenses in intraoperative OCT applications [S2].

## Sources
[S1] Refractive radiative transfer equation, https://doi.org/10.1145/2557605, fetched 2026-10-31
[S2] 3D fast ray-traced distortion correction of indirectly viewed intraoperative retinal OCT for surgical tool path reconstruction, https://europepmc.org/article/PMC/PMC13671697, fetched 2026-10-26
