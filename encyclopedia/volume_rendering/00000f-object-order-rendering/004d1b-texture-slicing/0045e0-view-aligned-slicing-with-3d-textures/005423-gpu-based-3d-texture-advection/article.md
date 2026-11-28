# GPU-based 3D Texture Advection

Also called: 3D IBFV, LIC-type visualization

## Overview
GPU-based 3D Texture Advection is a rendering technique that visualizes vector fields by advecting a texture pattern through a 3D volume using view-aligned slicing [S1]. It solves the problem of visualizing unsteady flow structures by maintaining a continuous texture field that evolves according to the flow velocity, rather than compositing discrete splats [S1]. It sets itself apart from its parent [View-Aligned Slicing with 3D Textures](870-view-aligned-slicing-with-3d-textures.md) by explicitly integrating the texture field over time using the vector field, and from its sibling [Texture Advection](18803-texture-advection.md) by operating on a 3D grid volume rather than a continuous polygonal mesh [S1].

## How it works
The method consists of three sequential steps: texture advection, texture injection/blending, and volume rendering [S1].

**Inputs and Outputs:**
*   **Input:** A 3D vector field $F(x, t)$ defined on a grid; a scalar or vector noise texture $I_t$ (injection texture); previous advected texture $T_{t-\Delta t}$; time step $\Delta t$.
*   **Output:** A rendered image of the volume, where opacity and color are modulated by the advected texture properties (e.g., divergence, curl) [S1].

**Data Structures:**
*   **Vector Field $F$:** Stored as a 3D texture or structured grid. In the tested implementation, datasets were $128^3$ [S1].
*   **Texture $T$:** A 3D texture storing the advected property field $\rho(x,t)$.
*   **Injection Texture $I_t$:** A 3D texture containing the noise or seed values to be advected.

**Algorithm Steps:**

1.  **Texture Advection:**
    The volume is advected slice-by-slice. For each point $x(t)$ in the current time step $t$, the source position $x(t-\Delta t)$ is calculated using the first-order Eulerian scheme [S1]:
    $$ x(t - \Delta t) = x(t) - \Delta t \cdot F(x(t), t) $$
    Here, $F(x(t), t)$ is the vector field evaluated at position $x(t)$ at time $t$, and $\Delta t$ is the integration step size [S1]. The value at $x(t-\Delta t)$ is sampled from the previous texture $T_{t-\Delta t}$ using tri-linear interpolation (as supported by the parent method) and stored as the advected component for $T_t$ [S1].

2.  **Texture Injection and Blending:**
    The newly advected texture is blended with an injection texture $I_t$ using multi-component weights $W_t$ and $V_t$ [S1]:
    $$ T_t = W_t \cdot T_{t-\Delta t} + V_t \cdot I_t $$
    Two specific settings for noise texture blending were tested [S1]:
    *   **Affine Combination:** An affine combination of values from $T_{t-\Delta t}$ is used to produce a fully opaque, Line Integral Convolution (LIC)-type volume. In this case, $V_t$ may be zero or negligible, and $W_t$ ensures normalization [S1].
    *   **Space-Variant Scalar Injection:** Space-variant scalar injection weights $V_t$ are applied, as described in 3D IBFV [23], allowing for variable opacity and feature highlighting [S1].

3.  **Property Evaluation and Rendering:**
    For each fragment rendered via view-aligned slicing, physical properties of the field are computed [S1].
    *   Properties such as velocity magnitude, determinant of the gradient tensor, magnitude of curl, helicity, and divergence are calculated [S1].
    *   Derivatives required for these properties (Equations 4-7 in the source) are approximated using central differencing (Equation 3 in the source) [S1].
    *   These properties are evaluated by the Multi-Dimensional Transfer Function (MDTF) to determine color and opacity [S1].
    *   The final image is generated using direct 3D texture-based volume rendering on the view-aligned slices [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gpu_based_3d_texture_advection(vector_field, noise_texture, prev_texture, dt, num_slices):
    """
    Performs GPU-based 3D texture advection for volume rendering.
    
    Args:
        vector_field: 3D array or texture representing F(x, t).
        noise_texture: 3D array or texture representing I_t.
        prev_texture: 3D array or texture representing T_{t-dt}.
        dt: Integration step size.
        num_slices: Number of view-aligned slices to render.
        
    Returns:
        rendered_image: The final composited image.
    """
    # Step 1: Texture Advection
    # Advect the volume slice by slice
    current_texture = empty_like(prev_texture)
    
    for each voxel x in volume:
        # Calculate source position using first-order Eulerian scheme
        # x(t-dt) = x(t) - dt * F(x(t), t)
        pos_current = x
        velocity = vector_field.sample(pos_current)
        pos_source = pos_current - dt * velocity
        
        # Sample from previous texture using tri-linear interpolation
        advected_value = prev_texture.trilinear_sample(pos_source)
        
        # Step 2: Texture Injection and Blending
        # T_t = W_t * T_{t-dt} + V_t * I_t
        # Example: Affine combination for LIC-type volume
        # W_t and V_t are multi-component weights
        # For affine combination, V_t is often 0 or small, W_t normalizes
        injection_value = noise_texture.sample(pos_current)
        
        # Settings 1: Affine combination (LIC-type)
        # W_t = 1.0, V_t = 0.0 (or small epsilon for stability)
        # Settings 2: Space-variant scalar injection (3D IBFV style)
        # W_t and V_t vary based on local flow properties or position
        
        weight_advect = 1.0 # Placeholder for specific W_t logic
        weight_inject = 0.0 # Placeholder for specific V_t logic
        
        current_texture[x] = weight_advect * advected_value + weight_inject * injection_value

    # Step 3: Volume Rendering with Property Evaluation
    rendered_image = render_volume(current_texture, vector_field, num_slices)
    
    return rendered_image

def render_volume(texture, vector_field, num_slices):
    """
    Renders the volume using view-aligned slices and MDTF.
    """
    image = zeros(width, height)
    
    for slice_index in range(num_slices):
        # Define view-aligned slice plane
        slice_plane = get_slice_plane(slice_index)
        
        for each fragment f on slice_plane:
            # Evaluate properties at fragment position
            pos = f.position
            velocity = vector_field.sample(pos)
            
            # Compute derivatives using central differencing
            # Approximations of Equations 4-7 using Equation 3
            grad_tensor = compute_gradient_tensor(vector_field, pos)
            curl = compute_curl(vector_field, pos)
            divergence = compute_divergence(vector_field, pos)
            helicity = compute_helicity(velocity, curl)
            
            # Evaluate Multi-Dimensional Transfer Function (MDTF)
            # Maps properties to color and opacity
            color, opacity = mdtf_evaluate(
                velocity_magnitude=norm(velocity),
                grad_det=determinant(grad_tensor),
                curl_mag=norm(curl),
                helicity=helicity,
                divergence=divergence
            )
            
            # Composite using standard volume rendering (e.g., front-to-back)
            # Note: The source mentions direct 3D texture-based rendering
            # which implies hardware-accelerated blending
            image = composite_fragment(image, color, opacity, f.depth)
            
    return image
```

## Subtopics

### [Texture Advection](18803-texture-advection.md)
Visualizes vector fields by advecting texture on a continuous polygonal mesh. Choose when geometric continuity is preferred over voxel-based storage.

### [GPU-based 3D Texture Advection with Logical-to-Physical Mapping](19774-gpu-based-3d-texture-advection-with-logical-to-physical-mapp.md)
Optimizes the above method by mapping logical 3D grid memory to physical 2D texture memory. Choose when 3D texture update performance is a bottleneck on the target GPU.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels and time steps, dominated by the texture sampling and blending operations [S1].
*   **Memory Complexity:** Requires storage for the vector field, the advected texture, and the injection texture. For a $128^3$ dataset, this is manageable on modern GPUs [S1].
*   **Performance:** Tested on steady (tornado, $128^3$) and unsteady (five jets, 2000 timesteps of $128^3$) datasets [S1]. The use of view-aligned slices leverages hardware-accelerated tri-linear interpolation [S1, 870-view-aligned-slicing-with-3d-textures.md].
*   **Strengths:** Produces LIC-type volumes that clearly show flow direction and structure; supports multi-dimensional transfer functions for rich property visualization [S1].
*   **Limitations:** First-order Eulerian integration can introduce numerical diffusion; 3D texture updates can be slow on some hardware (addressed by the child method) [S1, 19774-gpu-based-3d-texture-advection-with-logical-to-physical-mapp.md].

## Background
The technique was presented by Park, Budge, Linsen, Hamann, and Joy in 2004 [S1]. It builds on the system presented by Weiskopf and Ertl [22] and utilizes concepts from 3D IBFV [23]. The datasets were provided by the IDAV at the University of California, Davis [S1].

## Sources
[S1] Multi-dimensional Transfer Functions for Interactive 3D Flow Visualization, https://web.cs.ucdavis.edu/~hamann/ParkBudgeLinsenHamannJoy2004.pdf, fetched 2026-10-27
