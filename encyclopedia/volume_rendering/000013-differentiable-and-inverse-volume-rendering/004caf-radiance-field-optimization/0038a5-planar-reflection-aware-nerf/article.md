# Planar Reflection-Aware NeRF
Also called: Planar reflection-aware novel view synthesis

## Overview
Planar Reflection-Aware NeRF is a variant of Neural Radiance Fields that jointly models planar reflectors (e.g., windows) and explicitly casts reflected rays to capture the source of high-frequency reflections, preventing the creation of false geometry along the primary ray [S1]. It differs from standard NeRF by decomposing the reflection into a primary view and a reflected view, using sparse edge regularization to ensure the reflected color originates from the true scene geometry rather than duplicating artifacts at the reflection plane [S1].

## How it works
**Inputs:** Multi-view capture data (images and camera poses); detected planar reflectors (e.g., window planes) [S1].
**Outputs:** Optimized radiance field parameters, attenuation field parameters, and plane parameters [S1].

**Data Structures:**
1.  **Radiance Field:** A neural network (MLP) taking 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ as input, outputting volumetric density $\sigma$ and color $\mathbf{c}$ [S1].
2.  **Attenuation Field:** A neural network (or function) taking ray parameters as input, outputting an attenuation factor $A(\mathbf{r}')$ [S1].
3.  **Plane Parameters:** Defined by a point on the plane and a normal vector, used to compute intersections and reflections [S1].

**Numbered Steps:**
1.  **Detection:** Detect potential planar reflectors in the scene [S1].
2.  **Primary Ray Sampling:** For each camera ray $\mathbf{r}=(\mathbf{o}, \mathbf{d})$, sample 3D points and their viewing directions. Query the radiance field to obtain volumetric density $\sigma$ and color $\mathbf{c}$ [S1].
3.  **Intersection:** Determine if the primary ray intersects a detected planar reflector at point $\mathbf{x}$ [S1].
4.  **Reflection Direction:** If an intersection occurs, compute the reflected ray direction $\mathbf{d}'$ using the law of reflection [S1].
5.  **Reflected Ray Casting:** Cast a reflected ray $\mathbf{r}'$ from the intersection point $\mathbf{x}$ with direction $\mathbf{d}'$ [S1].
6.  **Reflected Field Query:** Query the *same* radiance field along the reflected ray $\mathbf{r}'$ to obtain density and color [S1].
7.  **Reflected Rendering:** Perform volume rendering along $\mathbf{r}'$ to compute the reflected color $\mathbf{C}(\mathbf{r}')$ [S1].
8.  **Attenuation:** Query the attenuation field $A(\mathbf{r}')$ to model the Fresnel effect, account for HDR tonemapping, and absorb errors due to intensity clipping [S1].
9.  **Composition:** Compute the final composited color $\mathbf{C}_{comp}(\mathbf{r})$ by adding the attenuated reflection to the primary color [S1]:
    $$
    \mathbf{C}_{comp}(\mathbf{r}) = \mathbf{C}(\mathbf{r}) + \mathbf{C}(\mathbf{r}') \cdot A(\mathbf{r}')
    $$
10. **Regularization:** Apply a sparse edge regularization loss $\mathcal{L}_{edge}$ between the primary color gradient $|\nabla \mathbf{C}(\mathbf{r})|$ and the reflected color gradient $||\nabla \mathbf{C}(\mathbf{r}')||$ to discourage false geometry [S1].
11. **Optimization:** Backpropagate the total loss (photometric and regularization) to update the radiance field, attenuation field, and plane parameters [S1].

**Formulas:**
*   **Law of Reflection:** The reflected direction $\mathbf{d}'$ is computed from the incident direction $\mathbf{d}$ and the plane normal $\mathbf{n}$ at intersection point $\mathbf{x}$ [S1]. The specific equation is referenced as Eq. (6) in the source, generally defined as $\mathbf{d}' = \mathbf{d} - 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n}$ [G] (standard vector reflection formula).
*   **Composited Color:** $\mathbf{C}_{comp}(\mathbf{r}) = \mathbf{C}(\mathbf{r}) + \mathbf{C}(\mathbf{r}') A(\mathbf{r}')$ [S1].
*   **Edge Regularization:** $\mathcal{L}_{edge}$ is applied between $|\nabla \mathbf{C}(\mathbf{r})|$ and $||\nabla \mathbf{C}(\mathbf{r}')||$ [S1]. The exact mathematical form of the loss function (e.g., L1, L2, or correlation) is not specified in the provided text [S1].

**Parameters:**
*   **Plane Detection Threshold:** Not given in the sources.
*   **Attenuation Field Capacity:** Not given in the sources.
*   **Regularization Weight:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Intensity Clipping:** The attenuation field is explicitly introduced to absorb errors due to intensity clipping in the input data [S1].
*   **False Geometry:** Without edge regularization, the method may create duplicate geometry along the primary ray at the same depth as the reflection [S1].
*   **HDR Tonemapping:** The attenuation field accounts for HDR tonemapping artifacts [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def planar_reflection_aware_nerf_render(ray_o, ray_d, planes, radiance_field, attenuation_field, samples_per_ray):
    """
    Renders a single ray with planar reflection awareness.
    
    Args:
        ray_o: Origin of the primary ray.
        ray_d: Direction of the primary ray.
        planes: List of detected planar reflectors (each defined by point and normal).
        radiance_field: Neural network returning (sigma, color) for (x, d).
        attenuation_field: Neural network returning attenuation factor A for ray r'.
        samples_per_ray: Number of samples along the ray.
        
    Returns:
        composited_color: The final color for the pixel.
    """
    # 1. Sample primary ray
    primary_points, primary_dirs = sample_ray(ray_o, ray_d, samples_per_ray)
    
    # 2. Query radiance field for primary ray
    primary_sigmas, primary_colors = radiance_field(primary_points, primary_dirs)
    
    # 3. Volume render primary color C(r)
    # Using standard alpha compositing (DVR)
    primary_color = volume_render(primary_sigmas, primary_colors)
    
    # 4. Check for intersection with planes
    intersection_point = None
    intersection_normal = None
    min_dist = infinity
    
    for plane in planes:
        dist = intersect_ray_plane(ray_o, ray_d, plane)
        if 0 < dist < min_dist:
            min_dist = dist
            intersection_point = ray_o + dist * ray_d
            intersection_normal = plane.normal
            
    # 5. If intersection found, handle reflection
    if intersection_point is not None:
        # Compute reflected direction d'
        # d' = d - 2(d . n)n
        reflected_dir = ray_d - 2 * dot(ray_d, intersection_normal) * intersection_normal
        
        # Cast reflected ray r' from intersection point
        reflected_ray_o = intersection_point
        reflected_ray_d = reflected_dir
        
        # Sample reflected ray
        reflected_points, reflected_dirs = sample_ray(reflected_ray_o, reflected_ray_d, samples_per_ray)
        
        # Query radiance field for reflected ray
        reflected_sigmas, reflected_colors = radiance_field(reflected_points, reflected_dirs)
        
        # Volume render reflected color C(r')
        reflected_color = volume_render(reflected_sigmas, reflected_colors)
        
        # Query attenuation field A(r')
        # Note: Attenuation field input details are not specified, assumed to take ray parameters
        attenuation = attenuation_field(reflected_ray_o, reflected_ray_d)
        
        # Compose color
        composited_color = primary_color + reflected_color * attenuation
    else:
        # No reflection, just primary color
        composited_color = primary_color
        
    return composited_color

def compute_loss(composited_color, ground_truth_color, primary_color, reflected_color):
    """
    Computes the total loss including photometric and edge regularization.
    """
    # Photometric loss
    l_photo = l1_loss(composited_color, ground_truth_color)
    
    # Edge regularization
    # Compute gradients of primary and reflected colors
    grad_primary = compute_gradient(primary_color)
    grad_reflected = compute_gradient(reflected_color)
    
    # Apply sparse edge regularization between |grad_primary| and ||grad_reflected||
    # Exact formula not specified in source [S1]
    l_edge = edge_regularization_loss(grad_primary, grad_reflected)
    
    total_loss = l_photo + l_edge
    return total_loss
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method requires additional ray casting and volume rendering for reflected rays, increasing computational cost compared to standard NeRF [S1].
*   **Memory Complexity:** Not given in the sources. The method introduces an additional attenuation field and plane parameters [S1].
*   **Strengths:** Accurately reconstructs high-frequency planar reflections without creating false geometry; handles HDR tonemapping and intensity clipping errors via the attenuation field [S1].
*   **Limitations:** Requires detection of planar reflectors; computationally more expensive due to reflected ray casting [S1].
*   **When to use:** Scenes with significant planar reflections (e.g., windows, mirrors) where standard NeRF fails by duplicating geometry [S1].
*   **When not to use:** Scenes without planar reflectors or with complex non-planar reflections [S1].

## Background
Planar Reflection-Aware NeRF was introduced to address the limitation of standard NeRF in handling complex planar reflections, which are often interpreted as erroneous scene geometries [S1]. It builds on the differentiable volume rendering framework [44-radiance-field-optimization.md] and extends it with explicit reflection modeling [S1]. The method was proposed in "Planar Reflection-Aware Neural Radiance Fields" [S1].

## Sources
[S1] Planar Reflection-Aware Neural Radiance Fields, https://doi.org/10.1145/3680528.3687560, fetched 2026-10-30
