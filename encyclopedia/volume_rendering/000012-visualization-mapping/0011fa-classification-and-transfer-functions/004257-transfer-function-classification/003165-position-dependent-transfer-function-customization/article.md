# Position-dependent Transfer Function Customization

Also called: Customization volume method, Interactive Position-dependent Customization of Transfer Function Classification Parameters, Interactive Position-dependent Customization via Primitives, Position-dependent customization, Seed Customization Points

## Overview
Position-dependent Transfer Function Customization is an interactive method for modifying transfer function classification parameters based on voxel position without preprocessing [S1]. It solves the problem of spatial inhomogeneities in volume data (e.g., scanner artifacts) by allowing users to define local adaptations of visualization parameters (such as intensity correction or color maps) at specific 3D locations [S1]. Unlike methods requiring preprocessing (e.g., distance fields or segmentation) or simple clipping that removes spatial context, this approach integrates a "customization volume" into the rendering pipeline to override or blend with the base transfer function, preserving surrounding structure while highlighting regions of interest [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Large volumetric dataset (scalar/vector fields), user-defined seed customization points (primitives), base transfer function.
*   **Output:** Rendered image with position-dependent parameter modifications.

**Data Structures:**
1.  **Seed Customization Point ($p$):**
    *   `position`: 3D voxel coordinates $(x, y, z)$.
    *   `primitive_type`: Enum (Point, Line, Rectangle, Cuboid).
    *   `extent`: 3D vector $(e_x, e_y, e_z)$ defining spatial size.
        *   $e_i = 0$ implies no extent in dimension $i$.
        *   Non-zero extent in 1, 2, or 3 dimensions defines Line, Rectangle, or Cuboid respectively [S1].
    *   `parameter_vector` ($\vec{v}$): Contains reference values for visualization parameters (e.g., float for intensity correction, or full color map) [S1].
    *   `id`: Unique integer identifier based on creation order [S1].
    *   `enabled`: Boolean flag. Disabled points are ignored during customization volume computation [S1].
2.  **Customization Volume:**
    *   A 3D texture/volume with configurable resolution $R \in [32^3, 512^3]$ voxels [S1].
    *   Stores interpolated parameter vectors for every voxel in the volume.
    *   Resolution is chosen based on available GPU memory and parameter complexity [S1].

**Numbered Steps:**

1.  **Define Seed Customization Points:**
    *   The user creates seed points using geometric primitives (points, lines, rectangles, cuboids) via 2D view interaction or voxel coordinates [S1].
    *   Each primitive is defined by its geometric center (position) and spatial extent per dimension [S1].
    *   The user assigns a `parameter_vector` $\vec{v}$ to each seed point [S1].
    *   *Note:* Individual specification for each voxel in large volumes ($>1000^3$) is not feasible; customization is limited to these seed points [S1].

2.  **Calculate Customization Volume:**
    *   Map the user-defined seed customization points into the smaller customization volume grid [S1].
    *   Interpolate parameter customizations for all other voxels in the customization volume using OpenGL compute shaders on the GPU [S1].
    *   **Interpolation Requirements:**
        1.  **Identity:** The interpolation must not change the value at the user-defined seed points [S1].
        2.  **Continuity/Monotonicity:** Interpolation between neighboring seed points must be continuous and monotonic [S1].
        3.  **Locality:** Each seed point should only have local influence [S1].
    *   Upon any position update or property adjustment, the customization volume is immediately recalculated [S1].

3.  **Integration into Volume Rendering Pipeline:**
    *   During the classification step of the rendering pipeline, gather input parameters (e.g., intensity, gradient) from the volume data at the current sampling position [S1].
    *   Fetch parameter modifications from the customization volume at the corresponding normalized position [S1].
    *   Apply modifications to the input parameters or the transfer function itself, depending on the parameter type [S1].
    *   *Example:* Lower opacity outside a region of interest while modifying colors inside it [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
class SeedPoint:
    position: Vec3  # Voxel coordinates
    extent: Vec3    # Spatial extent per dimension
    param_vector: Any  # e.g., float, ColorMap
    id: int
    enabled: bool

class CustomizationVolume:
    resolution: int  # e.g., 256
    data: Texture3D  # Stores interpolated param_vectors

# Step 1: User Interaction (Conceptual)
def add_seed_point(position, extent, param_vector):
    point = SeedPoint(position, extent, param_vector, next_id(), True)
    seed_points.append(point)
    next_id += 1
    recalculate_customization_volume()

# Step 2: GPU Interpolation
def recalculate_customization_volume():
    # Filter enabled points
    active_points = [p for p in seed_points if p.enabled]
    
    # Initialize customization volume texture
    # Resolution R is chosen between 32^3 and 512^3
    R = get_gpu_memory_appropriate_resolution()
    custom_vol = CustomizationVolume(R)
    
    # Dispatch compute shader to interpolate values
    # The shader iterates over all voxels in the customization volume
    # For each voxel, it calculates the interpolated parameter vector
    # based on the active seed points, satisfying:
    # 1. Value at seed points is preserved
    # 2. Interpolation is continuous and monotonic
    # 3. Local influence only
    dispatch_compute_shader(active_points, custom_vol)
    
    return custom_vol

# Step 3: Rendering Integration
def classify_voxel(volume_data, sampling_pos, custom_vol):
    # 1. Gather base parameters from volume data
    base_intensity = volume_data.get_value(sampling_pos)
    base_gradient = volume_data.get_gradient(sampling_pos)
    
    # 2. Fetch customization parameters from the customization volume
    # Normalize sampling_pos to [0, 1] for texture lookup
    normalized_pos = normalize(sampling_pos, volume_data.dimensions)
    custom_params = custom_vol.data.sample(normalized_pos)
    
    # 3. Apply modifications
    # Depending on parameter type, modify input or transfer function
    modified_intensity = apply_intensity_correction(base_intensity, custom_params.intensity_correction)
    
    # 4. Apply base transfer function with modified inputs
    color, opacity = transfer_function(modified_intensity, base_gradient)
    
    return color, opacity

# Visualization Feedback
def visualize_seed_points(seed_points, view):
    for point in seed_points:
        if point.enabled:
            # Option 1: Heat map based on a visualization parameter
            # Option 2: Unique ID coloring (creation order)
            # Points from same primitive share color
            draw_point(point.position, color=get_color_scheme(point))
            
            # Optional: Axis-aligned orientation lines to bounding box borders
            if show_orientation_lines:
                draw_lines_to_borders(point.position, point.extent)
```

**Parameters:**
*   **Customization Volume Resolution:** $32^3$ to $512^3$ voxels [S1].
*   **Primitive Extent:** Scalar values per dimension. $0$ in all directions = Point; non-zero in one = Line; two = Rectangle; three = Cuboid [S1].
*   **Primitive Position:** Geometric center, specified in voxel coordinates or via mouse in 2D views [S1].

**Edge Cases and Pitfalls:**
*   **Large Data Feasibility:** Individual user-driven specification for each voxel is not feasible for volumes $>1000^3$ voxels [S1].
*   **Precision:** Interpolation must strictly preserve values at seed points to ensure user intent is met [S1].
*   **Occlusion:** Seed points may be occluded in 3D views; they can be forced to foreground or visualized via 2D views [S1].

## Subtopics

### [Dual-Domain Interaction](5633-dual-domain-interaction.md)
Reverses the workflow by driving transfer function specification via direct interaction in the spatial domain (probing features) with feedback in the transfer function domain, solving the conceptual gap for multi-dimensional transfer functions.

## Operating characteristics
*   **Time Complexity:** The interpolation is performed on the GPU using compute shaders, achieving interactive frame rates during parameter customization [S1]. Updates to seed points trigger immediate recalculation [S1].
*   **Memory Complexity:** Memory requirements for the customization volume are configurable and do not depend on the size of the visualized data, making it applicable to large datasets (e.g., 9.5 GB) [S1].
*   **Performance Overhead:** The additional classification step has negligible performance overhead compared to the memory transfer bandwidth required for rendering large volume datasets [S1].
*   **Strengths:** No preprocessing required; preserves spatial context (unlike clipping); interactive feedback; scalable to large datasets [S1].
*   **Limitations:** Limited to axis-aligned primitives in the base implementation (non-axis-aligned planned for future work); relies on GPU compute shaders [S1].
*   **When to Use:** Interactive region-of-interest highlighting and parameter tuning in medical/biological research (e.g., CT scans) where spatial inhomogeneities exist [S1].
*   **When Not to Use:** When non-axis-aligned regions are critical and preprocessing is acceptable; when GPU compute shader support is unavailable [S1].

## Background
The method was proposed in 2015 by the authors of "Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering" [S1]. It addresses limitations in traditional transfer function classification, which assumes constant feature characteristics (e.g., linear intensity-to-HU mapping) [S1]. The approach extends concepts from segmentation and volume registration [S1] but avoids preprocessing. It is particularly relevant in medical and biological research for visualizing large datasets like CT scans of human hearts or mouse embryos [S1].

## Sources
[S1] Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering, https://doi.org/10.2312/vcbm.20151211, fetched 2026-10-29
