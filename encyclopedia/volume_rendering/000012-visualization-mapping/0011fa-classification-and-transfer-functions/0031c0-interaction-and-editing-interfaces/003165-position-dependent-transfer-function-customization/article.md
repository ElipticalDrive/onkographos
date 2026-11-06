# Position-dependent Transfer Function Customization

Also called: Customization volume method, Interactive Position-dependent Customization of Transfer Function Classification Parameters, Interactive Position-dependent Customization via Primitives, Position-dependent customization, Seed Customization Points

## Overview
This method enables interactive, position-dependent modification of arbitrary transfer function classification parameters in volume rendering by defining user-specified seed points and interpolating their effects across the volume [S1]. It solves the problem of spatial heterogeneity in volume data (e.g., scanner artifacts or varying tissue properties) where a single global transfer function fails to classify features correctly everywhere [S1]. Unlike [Automated Classification Widget Placement](5645-automated-classification-widget-placement.md) which automates widget location, or [Sketch-based Transfer Function Editing](5943-sketch-based-transfer-function-editing.md) which edits the function space directly, this method allows direct spatial manipulation of classification parameters via geometric primitives, integrating seamlessly into existing pipelines without preprocessing [S1].

## How it works
**Inputs:**
*   Volume dataset $D$ with dimensions $N_x \times N_y \times N_z$.
*   Base transfer function $TF$.
*   User-defined set of seed customization points $S = \{p_1, p_2, ..., p_n\}$.
*   Configuration for the customization volume resolution $R_{cv} \in [32, 512]^3$.

**Outputs:**
*   Rendered volume image with position-dependent parameter modifications.
*   Customization volume $C$ storing interpolated parameter vectors.

**Data Structures:**
1.  **Seed Customization Point ($p_i$):**
    *   `position`: 3D voxel coordinates $(x, y, z)$.
    *   `primitive_type`: Enum {Point, Line, Rectangle, Cuboid}.
    *   `extent`: 3D scalar vector $(e_x, e_y, e_z)$ defining spatial size.
        *   $e_x=e_y=e_z=0$: Point.
        *   One non-zero $e$: Line.
        *   Two non-zero $e$: Rectangle.
        *   Three non-zero $e$: Cuboid.
    *   `parameter_vector` ($\vec{v}$): Array of values representing modifications (e.g., float for intensity correction, or full color map).
    *   `id`: Unique integer identifier (creation order).
    *   `enabled`: Boolean flag.

2.  **Customization Volume ($C$):**
    *   3D texture/array of resolution $R_{cv} \times R_{cv} \times R_{cv}$.
    *   Each voxel stores a `parameter_vector` $\vec{c}$ of the same type and size as $\vec{v}$.

**Numbered Steps:**

1.  **Define Seed Customization Points:**
    The user creates seed points using geometric primitives (points, lines, rectangles, cuboids) via 2D view interaction or mouse in 3D views. Each primitive defines a region of influence. The position is the geometric center. The user assigns a `parameter_vector` $\vec{v}$ to each primitive. Disabled primitives are ignored in subsequent steps [S1].

2.  **Calculate Customization Volume:**
    The system computes the interpolation of $\vec{v}$ from the seed points to fill the `Customization Volume` $C$. This is performed on the GPU using OpenGL compute shaders to ensure interactive frame rates [S1].
    
    *Interpolation Requirements:*
    *   **Interpolation Requirement 1:** The interpolated value at a seed point's location must equal the user-defined $\vec{v}$ exactly.
    *   **Interpolation Requirement 2:** Interpolation between neighboring seed points must be continuous and monotonic.
    *   **Interpolation Requirement 3:** Each seed point has only local influence.
    
    The interpolation maps the seed points into the smaller volume $C$ (resolution $32^3$ to $512^3$). For each voxel in $C$, the shader calculates the weighted contribution of nearby seed primitives based on spatial distance and primitive extent, ensuring the constraints above are met [S1].

3.  **Integration into Volume Rendering Pipeline:**
    During ray casting, for each sampling position $\vec{s}$ in the volume:
    a. Gather input parameters from $D$ at $\vec{s}$ (e.g., intensity $I$, gradient $\nabla I$).
    b. Fetch the modification vector $\vec{c}$ from the `Customization Volume` $C$ at the normalized position corresponding to $\vec{s}$.
    c. Apply $\vec{c}$ to the input parameters or the transfer function $TF$ directly.
        *   If $\vec{c}$ modifies input parameters (e.g., intensity correction), adjust $I$ before passing to $TF$.
        *   If $\vec{c}$ modifies $TF$ (e.g., color map shift), apply the modification to $TF$ before evaluation.
    d. Compute final opacity and color using the modified $TF$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GPU Compute Shader Logic for Customization Volume Calculation
# Input: SeedPoints S, CustomizationVolumeResolution R
# Output: CustomizationVolume C

function calculate_customization_volume(S, R):
    # Initialize C with zeros or base values
    C = initialize_volume(R)
    
    # For each voxel in the customization volume
    for z in 0 to R.z-1:
        for y in 0 to R.y-1:
            for x in 0 to R.x-1:
                voxel_pos = (x, y, z)
                accumulated_params = empty_vector()
                total_weight = 0.0
                
                # Iterate over all enabled seed points
                for seed in S:
                    if not seed.enabled:
                        continue
                    
                    # Calculate distance/overlap between voxel and seed primitive
                    # Primitive is defined by center and extent
                    dist = distance_metric(voxel_pos, seed.primitive)
                    
                    # Weight function based on distance and extent
                    # Must satisfy: weight=1 at seed center, continuous, monotonic, local influence
                    weight = compute_weight(dist, seed.extent)
                    
                    if weight > 0:
                        accumulated_params += seed.parameter_vector * weight
                        total_weight += weight
                
                # Normalize to satisfy Interpolation Requirement 1 (value at seed point is preserved)
                # If total_weight is 0, keep base/default value
                if total_weight > 0:
                    C[x, y, z] = accumulated_params / total_weight
                else:
                    C[x, y, z] = default_params

# Rendering Step Integration
function render_volume(D, TF, C, ray):
    color = (0, 0, 0, 0)
    for t along ray:
        # 1. Gather input parameters from volume data
        intensity = D.sample(t)
        gradient = D.gradient(t)
        
        # 2. Fetch customization from volume
        # Map t to normalization [0,1] to index into C
        norm_t = normalize(t, D.dimensions)
        custom_params = C.sample(norm_t)
        
        # 3. Apply modifications
        # Example: Intensity correction
        modified_intensity = apply_intensity_correction(intensity, custom_params.intensity_correction)
        
        # Example: Color map shift
        modified_tf = apply_color_shift(TF, custom_params.color_shift)
        
        # 4. Evaluate Transfer Function
        opacity = modified_tf.evaluate(modified_intensity, gradient)
        
        # 5. Composite
        color = composite(color, opacity, modified_tf.color(modified_intensity))
        
    return color
```

**Parameters:**
*   **Customization Volume Resolution:** $32^3$ to $512^3$ voxels. Chosen based on GPU memory and required precision [S1].
*   **Primitive Extent:** Scalar per dimension. Defines the spatial reach of the seed point's influence [S1].

**Edge Cases and Pitfalls:**
*   **Large Data Sets:** Individual specification for each voxel in volumes $>1000^3$ is not feasible; seed points are mandatory [S1].
*   **Precision:** Interpolation must strictly preserve seed values (Requirement 1). Failure leads to visual artifacts at user-defined locations [S1].
*   **Occlusion:** Seed points may be occluded in 3D views. Visualization options include forcing points to foreground or using axis-aligned orientation lines to bounding box borders [S1].

## Subtopics

### [Dual-Domain Interaction](5633-dual-domain-interaction.md)
Reverses the workflow by allowing transfer function specification via direct spatial manipulation with immediate feedback in the TF domain, solving the conceptual gap between 3D geometry and abstract TF space.

## Operating characteristics
*   **Time Complexity:** The interpolation step is $O(N_{cv} \cdot N_{seeds})$ per frame update, where $N_{cv}$ is the number of voxels in the customization volume and $N_{seeds}$ is the number of seed points. Since $N_{cv}$ is small ($512^3$ max) and $N_{seeds}$ is small (user-defined), this is fast. The rendering overhead is negligible compared to the memory bandwidth required for large volume datasets [S1].
*   **Memory Complexity:** $O(R_{cv}^3 \cdot K)$, where $K$ is the size of the parameter vector. Independent of the input volume size $N_x N_y N_z$ [S1].
*   **Performance:** Achieves interactive frame rates during customization due to GPU compute shaders [S1].
*   **Strengths:** No preprocessing required; handles large datasets; preserves spatial context (unlike clipping); interactive feedback [S1].
*   **Limitations:** Limited to axis-aligned primitives in current implementation; interpolation complexity for non-linear parameter types (e.g., color maps) is non-trivial [S1].
*   **When to use:** Medical/biological research (CT scans) for region-of-interest highlighting and correcting local intensity inhomogeneities [S1].
*   **When not to use:** When global transfer functions are sufficient; when preprocessing (segmentation) is already available and preferred [S1].

## Background
The method was proposed in 2015 by researchers in the field of volume rendering visualization [S1]. It addresses limitations in traditional transfer function classification which assumes constant feature characteristics, a problem exacerbated by technical restrictions in scanner technology causing intensity homogeneities [S1]. It builds on prior work extending transfer function concepts to segmentation and registration [S1]. It is particularly applied in medical imaging, such as visualizing human hearts or mouse embryos, to highlight features like stenosis without losing surrounding context [S1].

## Sources
[S1] Interactive Position-dependent Customization of Transfer Function Classification Parameters in Volume Rendering, https://doi.org/10.2312/vcbm.20151211, fetched 2026-10-29
