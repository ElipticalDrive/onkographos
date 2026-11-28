# Point Light Shadow Shader


## Overview
The Point Light Shadow Shader is a local illumination method that computes shadows cast by a scene onto a horizontal reference plane from a single point light source [S1, unofficial]. It differs from general volume shadowing by explicitly intersecting shadow rays against a defined geometric plane (typically the ground) to determine occlusion, rather than integrating transmittance through a volumetric density field [S1, unofficial]. It distinguishes itself from hard-shadow methods by applying a Gaussian filter to the resulting shadow map to simulate soft shadows [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `rays`: A structure containing ray origins, directions, and maximum distances for the primary view rays.
*   `rb` (Render Buffer): A structure holding per-pixel data including hit status (`hit`), depth (`depth`), intersection coordinates (`xyz`), and normals (`normal`).
*   `pipeline`: An object providing access to a neural field tracer (`pipeline.nef`) and a ray tracing function (`pipeline.tracer`).
*   `point_light`: The 3D position of the point light source.
*   `min_y`: The Y-coordinate of the horizontal reference plane.
*   `rb.depth`: The current depth buffer values.

**Outputs:**
*   Updated `rb.rgb`: The final color buffer with shadowing applied.
*   Intermediate `shadow_map`: A 2D texture representing shadow intensity.

### Data Structures
*   **Rays**: Contains fields `origins` (tensor of shape $[N, 3]$), `dirs` (tensor of shape $[N, 3]$), `dist_min`, and `dist_max`.
*   **Render Buffer (rb)**:
    *   `hit`: Boolean tensor of shape $[N]$, indicating if a primary ray hit a surface.
    *   `depth`: Tensor of shape $[N, 1]$, storing the distance to the hit point.
    *   `xyz`: Tensor of shape $[N, 3]$, storing the 3D coordinates of the hit point.
    *   `normal`: Tensor of shape $[N, 3]$, storing the surface normal at the hit point.
    *   `shadow`: Tensor of shape $[N, 1]$, storing the shadow result (0.0 for shadowed, 1.0 for lit).
    *   `rgb`: Tensor of shape $[N, 3]$, storing the final RGB color.

### Steps
1.  **Plane Intersection Calculation**:
    Calculate the vertical distance from the ray origin to the reference plane:
    $$ \Delta = \text{rays.origins}[:, 1] - \text{min\_y} $$
    Compute the parameter $t$ along the ray direction to intersect the plane:
    $$ t_{\text{plane}} = \frac{\Delta}{\text{rays.dirs}[:, 1]} $$
    *Note: The source code implies division by the Y-component of the direction. If the direction Y is zero or near-zero, this step may require a guard against division by zero, though none is explicitly provided in the source [G].*

2.  **Initial Plane Hit Masking**:
    Identify rays that intersect the plane within a valid range:
    $$ \text{plane\_hit}_{\text{raw}} = (t_{\text{plane}} > 0) \land (t_{\text{plane}} < 500) $$

3.  **Refine Plane Hit**:
    Ensure the plane intersection is closer than any previously recorded surface hit (e.g., a wall or object standing on the floor). If the ray already hit an object closer than the plane, the plane hit is invalid:
    $$ \text{plane\_hit} = \text{plane\_hit}_{\text{raw}} \land (t_{\text{plane}} < \text{rb.depth}[:, 0]) $$
    Update the primary hit status to exclude these plane intersections from being treated as primary surface hits:
    $$ \text{rb.hit} = \text{rb.hit} \land \neg \text{plane\_hit} $$

4.  **Update Plane Intersection Data**:
    For pixels where `plane_hit` is true:
    *   Update depth:
        $$ \text{rb.depth}[\text{plane\_hit}] = t_{\text{plane}}[\text{plane\_hit}].\text{unsqueeze}(1) $$
    *   Update coordinates:
        $$ \text{rb.xyz}[\text{plane\_hit}] = \text{rays.origins}[\text{plane\_hit}] + \text{rays.dirs}[\text{plane\_hit}] \times t_{\text{plane}}[\text{plane\_hit}].\text{unsqueeze}(1) $$
    *   Set normal to the plane normal (upward):
        $$ \text{rb.normal}[\text{plane\_hit}] = [0, 1, 0] $$

5.  **Shadow Ray Generation**:
    *   **Origin**: Offset the intersection point slightly along the normal to prevent self-intersection artifacts:
        $$ \mathbf{o}_{\text{shadow}} = \text{rb.xyz} + 0.01 \times \text{rb.normal} $$
    *   **Direction**: Calculate the direction to the light source. The source code adds a small random perturbation to the light direction vector before normalization, likely to simulate a slightly diffuse light or reduce aliasing, though this is non-standard for a pure point light [S1, unofficial]:
        $$ \mathbf{d}_{\text{raw}} = \text{light\_o} - \mathbf{o}_{\text{shadow}} + \epsilon, \quad \epsilon \sim \mathcal{N}(0, 0.01) $$
        $$ \mathbf{d}_{\text{shadow}} = \text{normalize}(\mathbf{d}_{\text{raw}}) $$
    *   **Validity Check**: Ensure the light is in the direction of the normal (i.e., the light is "above" the surface relative to the normal):
        $$ \text{light\_hit} = (\mathbf{d}_{\text{shadow}} \cdot \text{rb.normal}) > 0 $$

6.  **Shadow Ray Tracing**:
    *   Construct `shadow_rays` with origins $\mathbf{o}_{\text{shadow}}$ and directions $\mathbf{d}_{\text{shadow}}$.
    *   Trace these rays against the neural field (`pipeline.nef`) using `pipeline.tracer`.
    *   Extract the hit status:
        $$ \text{rb.shadow}_{\text{raw}} = \text{pipeline.tracer}(\dots).\text{hit} $$

7.  **Shadow Computation**:
    *   Zero out shadow values for rays where the light was not in the valid direction (e.g., light below the floor):
        $$ \text{rb.shadow}[\neg \text{light\_hit}] = 0.0 $$
    *   Convert hit status to shadow intensity (1.0 if no hit, 0.0 if hit):
        $$ \text{shadow\_map}_{\text{raw}} = 1.0 - \text{rb.shadow}.\text{float}() $$
    *   Apply a bias/offset to soften the transition (value 0.7 added, then clamped):
        $$ \text{shadow\_map} = \text{clamp}(\text{shadow\_map}_{\text{raw}} + 0.7, 0.0, 1.0) $$

8.  **Soft Shadow Filtering**:
    Apply a Gaussian filter to the shadow map with $\sigma = 2$ to create soft shadows:
    $$ \text{shadow\_map}_{\text{final}} = \text{gaussian\_filter}(\text{shadow\_map}, \sigma=2) $$

9.  **Final Shading**:
    Multiply the base RGB color by the filtered shadow map:
    $$ \text{rb.rgb} = \text{rb.rgb} \times \text{shadow\_map}_{\text{final}} $$

### Parameters
*   `point_light`: List of 3 floats, default `[1.5, 4.5, 1.5]`.
*   `min_y`: Float, default `-2.0`.
*   `sigma`: Float, default `2` (for Gaussian filter).
*   `offset`: Float, default `0.01` (shadow ray origin offset).
*   `noise_std`: Float, default `0.01` (perturbation for shadow ray direction).
*   `bias`: Float, default `0.7` (added to shadow map before clamping).

### Edge Cases and Pitfalls
*   **Division by Zero**: If `rays.dirs[:, 1]` is zero, the plane intersection calculation $t_{\text{plane}}$ is undefined. The source does not explicitly handle this, which may lead to NaNs or infinities [G].
*   **Self-Shadowing**: The small offset (`0.01`) helps but may not be sufficient for all geometries, potentially causing acne or peter-panning.
*   **Light Direction Validity**: The check `(shadow_ray_d * rb.normal).sum(-1) > 0.0` ensures shadows are only computed for lights "above" the surface. If the light is below the plane, the shadow is forced to 0 (fully lit), which might be incorrect if the light is meant to illuminate from below.

## Operating characteristics
*   **Time Complexity**: Dominated by the shadow ray tracing step. If $N$ is the number of pixels, and $M$ is the number of samples per shadow ray, the complexity is $O(N \cdot M)$. The Gaussian filter adds $O(N)$ overhead with a constant kernel size.
*   **Memory Complexity**: $O(N)$ for storing ray data and shadow maps.
*   **Strengths**: Simple to implement; provides soft shadows via filtering; efficient for single point lights.
*   **Limitations**: Only works for a single point light; assumes a flat horizontal reference plane; soft shadows are achieved via post-processing blur rather than true geometric soft shadowing (which would require area light integration); perturbation of light direction is non-standard and may affect accuracy.

## Background
The method is derived from the `kaolin-wisp` project by NVIDIA GameWorks, which provides interactive neural field rendering tools [S1, unofficial]. It adapts standard ray-tracing techniques for neural fields to include simple shadowing for visualization purposes.

## Sources
[S1] NVIDIAGameWorks/kaolin-wisp, https://github.com/NVIDIAGameWorks/kaolin-wisp, fetched 2026-10-26 (unofficial)
