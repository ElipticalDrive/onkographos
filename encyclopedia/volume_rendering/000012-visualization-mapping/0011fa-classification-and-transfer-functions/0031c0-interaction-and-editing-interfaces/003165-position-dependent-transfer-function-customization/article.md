# Position-dependent Transfer Function Customization

Also called: Customization volume method, Interactive Position-dependent Customization via Primitives, Position-dependent customization, Seed Customization Points

## Overview
This method enables interactive, position-dependent modification of transfer function classification parameters in volume rendering by defining user-specified "seed customization points" and interpolating their effects across the volume via a "customization volume" [S1]. It solves the problem of spatial heterogeneity in data (e.g., scanner artifacts or varying tissue properties) where a single global transfer function fails to classify features uniformly [S1]. Unlike [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md) which optimizes widget layout, or [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md) which edits the function domain directly, this method operates in the spatial domain, allowing users to paint or place primitives in 3D space to locally override classification behavior without preprocessing [S1]. It differs from simple clipping by maintaining spatial context: it can lower opacity outside a region of interest while modifying colors inside, rather than discarding data [S1].

## How it works
The method consists of three main phases: defining seed points, calculating the customization volume, and integrating the result into the rendering pipeline.

### Inputs and Outputs
*   **Input:**
    *   Volume dataset $D$ with dimensions $W \times H \times D$.
    *   Set of seed customization points $S = \{p_1, p_2, ..., p_n\}$. Each point $p_i$ consists of:
        *   Position $\mathbf{x}_i$ (voxel coordinates).
        *   Spatial extent $\mathbf{e}_i$ (scalar values per axis defining a point, line, rectangle, or cuboid).
        *   Parameter vector $\mathbf{v}_i$ containing reference values for visualization parameters (e.g., intensity correction factors, color map entries) [S1].
    *   Base transfer function $T(\mathbf{u})$, where $\mathbf{u}$ are input parameters (intensity, gradient, etc.) [S1].
*   **Output:**
    *   Customization Volume $C_{vol}$ of resolution $R_c \times R_c \times R_c$ (typically $32^3$ to $512^3$) storing interpolated parameter modifications [S1].
    *   Modified optical properties (opacity $\alpha$, color $C$) for each sample point during ray casting [S1].

### Data Structures
1.  **Seed Customization Point:**
    *   `position`: `vec3` (voxel coordinates).
    *   `extent`: `vec3` (floats, 0.0 for point, >0.0 for line/rect/cuboid).
    *   `params`: `float[]` or structured data (e.g., color map) representing $\mathbf{v}_i$ [S1].
    *   `id`: Unique integer identifier (creation order) for visualization [S1].
    *   `active`: Boolean flag (disabled points are ignored in interpolation) [S1].

2.  **Customization Volume:**
    *   Texture format: 3D texture (e.g., `GL_RGBA32F` or similar high-precision format) [S1].
    *   Resolution: Configurable $R_c \in [32, 512]^3$ [S1].
    *   Content: Interpolated parameter vectors $\mathbf{v}(\mathbf{x})$ for every voxel $\mathbf{x}$ in the volume [S1].

### Algorithm Steps

**Step 1: Define Seed Customization Points**
The user defines seed points using geometric primitives (points, lines, rectangles, cuboids) via a UI or 2D views [S1].
*   Primitive position is the geometric center.
*   Extent defines size along axes. Extent $(0,0,0)$ is a point; one non-zero axis is a line; two is a rectangle; three is a cuboid [S1].
*   Each seed point $p_i$ is assigned a parameter vector $\mathbf{v}_i$ [S1].

**Step 2: Calculate Customization Volume**
The system interpolates the parameter vectors from the sparse seed points to fill the dense customization volume $C_{vol}$. This is performed on the GPU using OpenGL compute shaders to achieve interactive frame rates [S1].

*Interpolation Requirements:*
1.  **Interpolation Constraint:** The interpolated value at a seed point $p_i$ must equal $\mathbf{v}_i$ [S1].
2.  **Continuity and Monotonicity:** Interpolation between neighboring seed points must be continuous and monotonic [S1].
3.  **Local Influence:** Each seed point influences only a local region, preventing global artifacts [S1].

*Pseudocode for Interpolation (GPU Compute Shader):*
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for calculating customization volume voxel value
# Input: Seed points S, Customization Volume Resolution R_c, Volume Bounds B
# Output: Customization Volume Texture C_vol

def calculate_customization_volume(S, R_c, B):
    # Initialize C_vol with default/base values (e.g., identity or base TF params)
    C_vol = initialize_texture(R_c, default_value=BASE_PARAMS)
    
    # For each voxel in the customization volume
    for x in range(R_c):
        for y in range(R_c):
            for z in range(R_c):
                # Map voxel index to world/voxel coordinates
                world_pos = index_to_world(x, y, z, R_c, B)
                
                # Gather contributions from all active seed points
                weighted_sum = ZERO_VECTOR
                weight_total = 0.0
                
                for seed in S:
                    if not seed.active:
                        continue
                    
                    # Check if world_pos is within the seed's primitive extent
                    if is_inside_primitive(world_pos, seed.position, seed.extent):
                        # Calculate distance-based weight
                        # Note: Exact distance metric not specified, but must satisfy
                        # local influence and continuity [S1]. 
                        # Common approach: Inverse distance or Gaussian falloff within extent.
                        dist = distance(world_pos, seed.position)
                        w = compute_weight(dist, seed.extent)
                        
                        if w > 0:
                            weighted_sum += w * seed.params
                            weight_total += w
                
                # Normalize
                if weight_total > 0:
                    C_vol[x, y, z] = weighted_sum / weight_total
                else:
                    # If no seed influences this voxel, keep base/default value
                    C_vol[x, y, z] = BASE_PARAMS
                    
    return C_vol
```
*Note: The exact interpolation function (e.g., linear, cubic, inverse distance) is not explicitly defined in the source, but must satisfy the three requirements: exact match at seeds, continuity/monotonicity, and local influence [S1].*

**Step 3: Integration into Volume Rendering Pipeline**
The customization volume is integrated into the classification step of the ray casting pipeline [S1].

1.  **Sampling:** Ray casting samples the volume dataset $D$ at position $\mathbf{x}$.
2.  **Input Parameter Gathering:** Gather base input parameters $\mathbf{u}$ (e.g., intensity $f(\mathbf{x})$, gradient $\nabla f(\mathbf{x})$) [S1].
3.  **Customization Fetch:** Sample the customization volume $C_{vol}$ at the corresponding normalized position of $\mathbf{x}$ to retrieve the modification vector $\Delta \mathbf{v}$ [S1].
4.  **Parameter Modification:** Apply $\Delta \mathbf{v}$ to $\mathbf{u}$ or the transfer function $T$ directly.
    *   *Example:* If $\mathbf{v}$ represents an intensity gamma correction, modify the intensity $f(\mathbf{x})$ before passing it to $T$.
    *   *Example:* If $\mathbf{v}$ represents a color map, modify the color output of $T$ [S1].
5.  **Classification:** Compute final opacity $\alpha$ and color $C$ using the modified parameters [S1].
6.  **Compositing:** Perform standard volume compositing (e.g., front-to-back) [S1].

### Parameters
*   **Customization Volume Resolution ($R_c$):** $32^3$ to $512^3$ voxels. Chosen based on GPU memory and desired precision [S1].
*   **Primitive Extent:** Scalar values per axis. Defines the spatial influence of a seed point [S1].
*   **Parameter Vector ($\mathbf{v}$):** Can be simple floats (e.g., intensity shift) or complex structures (e.g., full color map) [S1].

### Edge Cases and Pitfalls
*   **Large Data Sets:** Individual specification for each voxel in volumes $>1000^3$ is infeasible; hence the seed-based approach is required [S1].
*   **Precision:** High-precision textures (e.g., 32-bit float) are recommended for the customization volume to avoid quantization artifacts in parameter modifications [S1].
*   **Occlusion:** Seed points may be occluded in 3D views. Visualization options include forcing points to foreground or using axis-aligned orientation lines to bounding box borders [S1].
*   **Performance:** The additional classification step adds negligible overhead compared to memory transfer bandwidth for large datasets [S1].

## Subtopics

### [Dual-Domain Interaction](5633-dual-domain-interaction.md)
An interaction paradigm where users probe features in the spatial domain to set transfer function parameters, with visual feedback in the transfer function domain. It addresses the conceptual gap between 3D geometry and abstract TF space.

## Operating characteristics
*   **Time Complexity:** Interpolation is performed per voxel in the customization volume ($O(R_c^3 \cdot N)$ where $N$ is number of seeds). Rendering adds a single texture lookup per sample point. Overall overhead is negligible compared to volume data transfer [S1].
*   **Memory Complexity:** Additional memory is $O(R_c^3 \cdot P)$ where $P$ is parameter vector size. Independent of original volume size $W \times H \times D$ [S1].
*   **Performance:** Interactive frame rates are achievable due to GPU-based interpolation and small customization volume size [S1].
*   **Strengths:** No preprocessing required; handles large datasets; maintains spatial context; interactive feedback [S1].
*   **Limitations:** Limited to axis-aligned primitives (in current implementation); interpolation artifacts if seed points are sparse or conflicting; requires GPU compute shaders [S1].
*   **When to Use:** Medical/biological research (e.g., CT scans) for region-of-interest highlighting and correcting local intensity inhomogeneities [S1].
*   **When Not to Use:** When global classification is sufficient; when preprocessing (segmentation) is already available and preferred [S1].

## Background
The method was proposed in 2015 by [S1] to address intensity homogeneities in volume data caused by scanner technology restrictions [S1]. It extends traditional transfer function concepts by allowing position-dependent customization without preprocessing, building on earlier work in classification and segmentation [S1]. It is particularly relevant in medical imaging (e.g., CT scans of human hearts) where local variations in tissue density or scanner artifacts require localized adjustments [S1].

## Sources
[S1] Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering, https://doi.org/10.2312/vcbm.20151211, fetched 2026-10-29
