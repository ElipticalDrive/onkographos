# Vortex Flow Function

Also called: Helical Path Animation, Spiral Vortex Function, helical turbulence, swirling gas animation, vortex function

## Overview
The Vortex Flow Function generates gas animation by advecting sample points through a static noise space along a helical or rotational path, optionally modified by turbulence [S1, unofficial]. It solves the problem of creating convincing swirling, rising, or rolling gas effects (such as steam or smoke) without the computational cost of full fluid simulation [S1, unofficial]. It differs from its siblings, such as [Spot Noise](2959-spot-noise.md) or [Local Cloud Flow with FlowMap](2258-local-cloud-flow-with-flowmap.md), by explicitly defining a geometric flow field (helical or rotational) that displaces the evaluation point, rather than relying on pre-computed vector fields or localized spot projections [S1, unofficial].

## How it works
The method consists of two primary variants: the **Helical Path Animation** (for continuous rising/rolling gas) and the **Vortex Function** (for localized swirling disturbances). Both rely on the parent's principles of domain definition, value assignment, and domain warping [S1, S4, S5, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `pnt`: The current 3D sample point in the noise space.
    *   `frame_num`: The current animation frame index.
    *   `ff->center`: The 3D center of the vortex.
    *   `ff->axis`: The 3D axis of rotation for the vortex.
    *   `ff->distance`: The maximum radius of the vortex influence.
    *   `ff->falloff_start`: The distance inside which the vortex has full strength.
    *   `ff->parms`: An array of parameters controlling speed, falloff, and timing.
    *   `vol.shape`: Volume shape information (center, inverse radius squared).
*   **Outputs**:
    *   `density`: The final scalar density value.
    *   `path`: The displacement vector (direction) for advection.
    *   `percent_to_use`: The falloff factor applied to the density.
    *   `velocity`: The speed multiplier for the flow.

### Data Structures
*   **Precomputed Tables**:
    *   `pow_table[POW_TABLE_SIZE]`: Maps turbulence values to density using a power function.
    *   `ramp[RAMP_SIZE]`: Maps radial distance from the volume center to a falloff factor.
    *   `offset[OFFSET_SIZE]`: Pseudo-random offsets for vertical and radial falloff variation.
*   **Static State** (computed once per frame or initialization):
    *   `theta`: Rotation angle for helical path.
    *   `down`: Vertical offset for helical path.
    *   `cos_theta2`, `sin_theta2`: Precomputed trigonometric components for the helical path.

### Steps: Helical Path Animation
This variant creates upward or downward swirling motion by moving the sample point along a helix.

1.  **Initialization** (performed once or when `frame_num` changes):
    *   Compute `theta = (frame_num % SWIRL_FRAMES) * SWIRL`.
    *   Compute `down = frame_num * DOWN * 3.0 + 4.0`.
    *   Compute `cos_theta2 = RAD1 * cos(theta) + 2.0`.
    *   Compute `sin_theta2 = RAD2 * sin(theta) - 2.0`.
    *   Populate `pow_table`: For `i` from `POW_TABLE_SIZE-1` down to `0`:
        $$ \text{pow\_table}[i] = \left( \frac{i}{\text{POW\_TABLE\_SIZE}-1} \times \text{parms}[1] \times 2.0 \right)^{\text{parms}[2]} $$
    *   Populate `ramp` table using `make_ramp_table(ramp)`.

2.  **Calculate Direction Vector**:
    *   Compute noise perturbation: `tmp = fast_noise(pnt)`.
    *   Compute direction components:
        $$ \text{direction}.x = \text{pnt}.x + \text{cos\_theta2} + \text{tmp} $$
        $$ \text{direction}.y = \text{pnt}.y - \text{down} + \text{tmp} $$
        $$ \text{direction}.z = \text{pnt}.z + \text{sin\_theta2} + \text{tmp} $$

3.  **Evaluate Turbulence**:
    *   Compute `turb = fast_turbulence(direction)`.

4.  **Map to Density**:
    *   Compute index: `indx = (int)(turb * 0.5 * (POW_TABLE_SIZE - 1))`.
    *   Clamp `indx` to `[0, POW_TABLE_SIZE - 1]`.
    *   Set `*density = pow_table[indx]`.

5.  **Apply Radial Falloff**:
    *   Compute vector from volume center: `XYZ_SUB(diff, vol.shape.center, pnt_world)`.
    *   Compute squared distance: `dist_sq = DOT_XYZ(diff, diff)`.
    *   Compute normalized squared distance: `density_max = dist_sq * vol.shape.inv_rad_sq.y`.
    *   Add random offset: `indx = (int)((pnt.x + pnt.y + pnt.z) * 100) & (OFFSET_SIZE - 1)`.
    *   `density_max += parms[3] * offset[indx]`.
    *   If `density_max >= 0.25`:
        *   Compute ramp index: `i = (density_max - 0.25) * 4 / 3 * RAMP_SIZE`.
        *   Clamp `i` to `[0, RAMP_SIZE - 1]`.
        *   `density_max = ramp[i]`.
        *   `*density *= density_max`.

6.  **Apply Vertical Falloff**:
    *   Compute vertical distance: `dist = pnt_world.y - vol.shape.center.y`.
    *   If `dist > 0.0`:
        *   Normalize: `dist = (dist + offset[indx] * 0.1) * vol.shape.inv_rad.y`.
        *   If `dist > 0.05`:
            *   Compute offset factor: `offset2 = (dist - 0.05) * 1.111111`.
            *   Apply exponential falloff: `offset2 = 1 - (exp(offset2) - 1.0) / 1.718282`.
            *   Scale by parameter: `offset2 *= parms[1]`.
            *   `*density *= offset2`.

### Steps: Vortex Function
This variant creates a localized swirling disturbance by rotating the point around an axis.

1.  **Calculate Distance from Center**:
    *   Compute difference vector: `XYZ_SUB(diff, *pt, ff->center)`.
    *   Compute distance: `dist = sqrt(DOT_XYZ(diff, diff))`.
    *   Compute normalized distance squared: `dist2 = dist / ff->distance`.

2.  **Calculate Rotation Angle**:
    *   Compute angle `theta` based on frame number and distance:
        $$ \theta = \frac{\text{ff->parms}[0] \times (1 + 0.001 \times \text{frame\_num})}{\text{pow}(0.1 + \text{dist2} \times 0.9, \text{ff->parms}[1])} $$
    *   *Note*: A more physically realistic variant uses conservation of angular momentum: $\theta = C / r$, where $r$ is the distance from the center [S1, unofficial].

3.  **Transform Point**:
    *   Calculate rotation matrix `mat` about `ff->axis` using `theta`: `calc_rot_mat(theta, ff->axis, mat)`.
    *   Transform point: `transform_XYZ(1, mat, pt, &pt2)`.

4.  **Calculate Path Vector**:
    *   Compute direction: `XYZ_SUB(dir, pt2, *pt)`.
    *   Assign to output: `path->x = dir.x; path->y = dir.y; path->z = dir.z;`.

5.  **Calculate Falloff and Velocity**:
    *   Compute `ratio_mult` based on frame timing:
        *   If `frame_num < ff->parms[4]`: `ratio_mult = 0`.
        *   Else if `frame_num <= ff->parms[5]`:
            $$ \text{ratio\_mult} = \frac{\text{frame\_num} - \text{ff->parms}[4]}{\text{ff->parms}[5] - \text{ff->parms}[4]} \times \text{ff->parms}[2] $$
        *   Else: `ratio_mult = ff->parms[2]`.
    *   Determine `percent_to_use` and `velocity`:
        *   If `dist > ff->distance`:
            *   `*percent_to_use = 0`.
            *   `*velocity = 1`.
        *   Else if `dist < ff->falloff_start`:
            *   `*percent_to_use = 1.0 * ratio_mult`.
            *   `*velocity = 1.0 + (1.0 - (dist / ff->falloff_start))`.
        *   Else:
            *   Compute `d2 = (dist - ff->falloff_start) / (ff->distance - ff->falloff_start)`.
            *   `*percent_to_use = (\cos(d2 \times \pi) + 1.0) \times 0.5 \times \text{ratio\_mult}`.
            *   `*velocity = 1.0 + (1.0 - (dist / ff->falloff_start))`.

### Parameters
*   `SWIRL_FRAMES`: Number of frames for one full 360-degree swirl cycle [S1, unofficial].
*   `RAD1`, `RAD2`: Constants determining the elliptical shape of the swirling path [S1, unofficial].
*   `ff->parms[0]`: Coefficient for the rotation angle calculation (base speed) [S1, unofficial].
*   `ff->parms[1]`: Exponent for the distance-based rotation angle falloff [S1, unofficial].
*   `ff->parms[2]`: Maximum strength of the vortex [S1, unofficial].
*   `ff->parms[4]`: Frame number at which the vortex strength begins to increase [S1, unofficial].
*   `ff->parms[5]`: Frame number at which the vortex strength reaches its maximum [S1, unofficial].
*   `DOWN`: Vertical step size per frame for helical path [S1, unofficial].
*   `SWIRL`: Constant for helical rotation step [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: The use of `frame_num` in angle calculations can lead to precision issues for very large frame numbers; modulo operations or periodic resets are recommended.
*   **Boundaries**: The radial falloff uses a hard cutoff at `ff->distance` for the vortex function, but a smooth cosine falloff for the transition zone. Ensure `ff->falloff_start < ff->distance`.
*   **Degenerate Input**: If `dist` is zero in the vortex function, the rotation is undefined; however, the code handles this by rotating around the axis, which results in no displacement if the point is on the axis.

## Subtopics

### [Smoke Column with Helical Displacement](2743-smoke-column-with-helical-displacement.md)
This method generates a single column of rising smoke by deforming a vertical cylinder, simulating dispersion, turbulence, and swirling motion. It differs from its parent, the [Vortex Flow Function](2538-vortex-flow-function.md), by explicitly combining vertical helical displacement for bending/swirling with height-dependent density decay and turbulence scaling to simulate the physical dispersion of a smoke plume. It is distinct from [Analytical Vector Fields](3124-analytical-vector-fields.md) by focusing on a specific procedural geometry deformation (cylinder to helical column) rather than general field integration.

### [Analytical Vector Fields](3124-analytical-vector-fields.md)
Analytical vector fields generate flow data using explicit mathematical functions of position and time, rather than relying on pre-computed simulation grids or particle systems. This approach allows for real-time evaluation of the velocity at any point in space, making it suitable for procedural volume rendering where memory bandwidth for large simulation grids is a bottleneck. Unlike simulation-based methods, analytical fields are defined by closed-form equations, which distinguishes them from their siblings that may combine procedural noise with physical constraints or specific geometric displacements.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per sample point, as it involves a fixed number of arithmetic operations and table lookups.
*   **Memory Complexity**: $O(1)$ auxiliary memory for precomputed tables (`pow_table`, `ramp`, `offset`), which are small and static.
*   **Performance**: Highly efficient for real-time rendering due to the lack of iterative solvers or large grid lookups. Performance is limited by the cost of the underlying `fast_noise` and `fast_turbulence` functions.
*   **Strengths**: Simple to implement, visually appealing for gas-like effects, easily animatable via `frame_num`.
*   **Limitations**: Not a physically accurate simulation of fluid dynamics; the "conservation of angular momentum" variant is an approximation. The helical path is rigid and may not adapt to complex scene geometry.
*   **When to Use**: Ideal for stylized gas effects (steam, smoke, fog) where performance is critical and physical accuracy is secondary.
*   **When Not to Use**: When realistic fluid dynamics, interaction with obstacles, or complex turbulence cascades are required.

## Background
The Vortex Flow Function and Helical Path Animation were introduced by David Ebert in his work on procedural modeling, animation, and rendering of gases and fluids [S1, unofficial]. The techniques are based on earlier work by Karl Sims, who used similar helical paths for moving marble and other procedural effects [S1, unofficial]. The method leverages the simplicity of polar coordinate transformations extended to 3D to create the illusion of complex fluid motion.

## Sources
[S1] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
