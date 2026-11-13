# SDF Transformations
Also called: SDF Composition, SDF Modifiers

## Overview
SDF Transformations modify the spatial domain of a Signed Distance Field (SDF) by applying geometric transformations to the query point $\mathbf{p}$ before evaluating the primitive's distance function. This technique allows for the creation of complex procedural geometry, such as tiling, twisting, and dynamic animation, without altering the underlying primitive definitions or requiring mesh processing [S1, unofficial][S2]. Unlike static composition which combines primitives via Boolean operations, transformations map the world space into the local space of the primitive, enabling effects like repetition and deformation while preserving the Lipschitz condition required for valid sphere tracing [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Query point $\mathbf{p}$ (world space), transformation parameters (e.g., spacing, twist angle, time $t$), and the base SDF function $f_{base}$.
*   **Output:** Transformed distance value $d = f_{base}(\mathbf{p}')$, where $\mathbf{p}'$ is the transformed point.

**Data Structures:**
No specific data structures are required beyond the transformation parameters and the base SDF function. For dynamic operators, a Holistic Scene Dynamics (HSD) function may be used to generate transformation matrices based on time and object ID [S2].

**Steps:**
1.  **Identify Transformation Type:** Determine if the transformation is static (e.g., repetition, deformation) or dynamic (e.g., time-varying animation) [S1, unofficial][S2].
2.  **Compute Transformed Point $\mathbf{p}'$:** Apply the inverse of the desired geometric transformation to the query point $\mathbf{p}$.
    *   For **Repetition (Tiling)**: Apply modular arithmetic to wrap the space.
        $$ \mathbf{p}' = \text{mod}(\mathbf{p}, \mathbf{s}) - \frac{\mathbf{s}}{2} $$
        where $\mathbf{s}$ is the spacing vector [S1, unofficial].
    *   For **Deformation**: Apply a non-linear function to the position.
        $$ \mathbf{p}' = \text{twist}(\mathbf{p}) $$
        where $\text{twist}$ rotates the point around an axis based on its height or distance [S1, unofficial].
    *   For **Dynamic Operators**: Calculate a dynamic variation using the Holistic Scene Dynamics Function $f_{HSD}(\mathbf{d}, t)$, where $\mathbf{d}$ is object input data and $t$ is simulation time. This may generate a transformation matrix $M$ [S2].
3.  **Evaluate Base SDF:** Compute the distance using the transformed point.
    $$ d = f_{base}(\mathbf{p}') $$
4.  **Dynamic Ray Transformation (Optional):** In dynamic instancing pipelines, apply an additional transformation to the position on the ray from the camera to the SDF component at each iteration of the ray marching algorithm to preserve the correct form of the SDF during animation [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_transform(p, transform_type, params, f_base):
    """
    Apply a transformation to point p before evaluating f_base.
    
    Args:
        p: vec3, query point in world space
        transform_type: str, 'repetition', 'deformation', 'dynamic'
        params: dict, transformation parameters
        f_base: callable, base SDF function
        
    Returns:
        float, transformed distance
    """
    if transform_type == 'repetition':
        spacing = params['spacing']
        # p' = mod(p, spacing) - spacing/2
        p_prime = mod(p, spacing) - spacing / 2.0
        return f_base(p_prime)
    
    elif transform_type == 'deformation':
        # Example: twist deformation
        angle = params['angle']
        axis = params['axis'] # e.g., vec3(0, 1, 0)
        # p' = twist(p)
        p_prime = twist(p, angle, axis)
        return f_base(p_prime)
    
    elif transform_type == 'dynamic':
        # Dynamic Operators: use HSD function
        # f_HSD(d, t) -> transformation matrix M
        object_id = params['object_id']
        time = params['time']
        d = object_id
        M = holistic_scene_dynamics(d, time)
        
        # Apply inverse transformation to p
        # p' = M^-1 * p (assuming homogeneous coordinates or 3x3 rotation/scaling)
        p_prime = inverse(M) * p
        return f_base(p_prime)
    
    else:
        return f_base(p)

def twist(p, angle, axis):
    """Simple twist deformation around axis."""
    # Implementation depends on specific twist function
    # Placeholder for actual twist logic
    return p # [G] Placeholder: Replace with actual twist formula
```

**Parameters:**
*   `spacing`: Vector defining the repetition interval [S1, unofficial].
*   `angle`: Float defining the twist intensity [S1, unofficial].
*   `time`: Float for dynamic variations [S2].
*   `object_id`: Identifier for unique dynamic variations [S2].

**Edge Cases and Pitfalls:**
*   **Lipschitz Condition:** Transformations must preserve the Lipschitz constant of 1. Non-linear transformations like twisting can violate this if not carefully implemented, leading to tunneling in sphere tracing [S2, 9643].
*   **Precision:** Modular arithmetic for repetition can suffer from precision issues at large scales.
*   **Dynamic Instancing:** Dynamic operators require calculating the grid cell vector independently from the Instancing operator to ensure correct transformation application [S2].

## Subtopics

### Dynamic Operators
Dynamic Operators extend the Global operators collection by applying unique affine transformations and material animations to 3D objects using the Holistic Scene Dynamics Function [S2]. Unlike original Global operators that used the Instancing operator's output (grid cell vector) as input, Dynamic Operators calculate the grid cell vector independently and transform the ray before applying the Instancing operator [S2]. This allows for time-varying effects such as wind sway or object translation based on simulation time and object ID [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per SDF evaluation, as transformations are simple arithmetic operations.
*   **Memory Complexity:** $O(1)$, as no additional data structures are needed beyond transformation parameters.
*   **Strengths:** Extremely low memory footprint; procedural geometry is defined by functions, not data structures [S1, unofficial]. Allows for complex effects like tiling and deformation without mesh processing [S1, unofficial].
*   **Limitations:** Transformations must preserve the Lipschitz condition to ensure valid sphere tracing [S2]. Dynamic operators require additional processing passes and independent grid cell calculation [S2].
*   **When to Use:** When creating repetitive patterns, deformed shapes, or animated procedural scenes without increasing memory usage [S1, unofficial][S2].

## Background
SDF Transformations are a fundamental technique in procedural ray marching, allowing for the creation of complex scenes from simple primitives [S1, unofficial]. The concept of applying transformations to the query point before evaluation is standard in SDF rendering [S1, unofficial]. Dynamic Operators were introduced as an extension to the Holistic UOI rendering pipeline to enable dynamic variations of 3D objects using the Holistic Scene Dynamics Function [S2].

## Sources
[S1] ray marching vs ray tracing, https://tutorials.tektite.studio/ray-marching/ray-marching-vs-ray-tracing/, fetched 2026-10-27 (unofficial)
[S2] Real-time visualization of Dynamic Unlimited Objects Instancing, https://doi.org/10.24132/csrn.2018.2802.4, fetched 2026-10-26
