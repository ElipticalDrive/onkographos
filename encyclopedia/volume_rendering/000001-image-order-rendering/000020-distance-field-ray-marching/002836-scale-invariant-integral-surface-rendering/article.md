# Scale-Invariant Integral Surface Rendering

Also called: SCALIS rendering, method using homothetic distance and quadratic polynomial interpolation

## Overview
Scale-Invariant Integral Surface Rendering (SCALIS) is a ray-marching algorithm designed to efficiently render integral surfaces defined by skeletal primitives with varying radii [S1]. It solves the performance bottleneck of standard sphere tracing, specifically the excessive number of steps required along grazing rays where the distance field provides poor step size estimates [S1]. By combining a homothetic distance-based initial sampling strategy with quadratic polynomial interpolation of the field, it reduces computation time by up to 70% compared to previous methods while maintaining visual fidelity [S1].

## How it works
The method renders scale-invariant integral surfaces defined by skeletons (random or procedural) and associated radius functions [S1].

**Inputs:**
*   Skeleton definition (points and connectivity).
*   Radius function or kernel parameters defining the scale-invariant integral surface [S1].
*   Camera parameters (origin, direction, near/far planes) [S1].

**Outputs:**
*   Rendered image with color and depth [S1].

**Data Structures:**
*   **Skeleton Primitives:** Defined by sphere-cone supports. Each primitive has an axis-aligned bounding box (AABB) computed from its support [S1].
*   **A-Buffer:** A GPU-based data structure used to manage primitive visibility and overlap during ray processing [S1].
*   **Heap:** A dynamic structure storing segment primitives whose supports overlap the initial intervals defined by the homothetic distance minimum [S1].

**Algorithm Steps:**

1.  **Preprocessing (CPU):**
    *   Generate skeletons and compute radius parameters [S1].
    *   For each model, compute the AABB for every primitive based on its sphere-cone support [S1].

2.  **Ray Initialization (GPU):**
    *   Launch sets of rays in six directions (positive and negative orientations along the three main axes) [S1].
    *   Sample ray origins uniformly on the faces of the model's bounding box [S1].
    *   Target a total of 10 million rays per model for statistical validation, though production uses adaptive counts [S1].

3.  **Ray Marching Loop:**
    *   **Homothetic Distance Evaluation:** For each ray, compute the homothetic distance to segment primitives. This metric determines the initial sampling strategy and step sizes [S1].
    *   **Heap Management:** Maintain a heap of active primitives. The heap must accommodate all primitives whose supports overlap the initial intervals defined by the argument of the minimum of the homothetic distance [S1].
    *   **Field Evaluation:** Evaluate the field value and its gradient simultaneously. This allows the computation of the directional derivative at a cost of less than twice a single field evaluation [S1]. Segment parameters are fetched once from memory for both values [S1].
    *   **Quadratic Interpolation:** Use quadratic polynomial interpolation on the field values to estimate the surface crossing more accurately than linear methods, reducing the number of steps [S1].
    *   **Step Advancement:** Advance the ray based on the interpolated surface estimate and the homothetic distance bounds [S1].
    *   **Termination:** Stop if the surface is hit (field value crosses zero) or the ray escapes the scene bounds [S1].

4.  **Shading:**
    *   Compute surface normals using the gradient of the field at the hit point [S1].
    *   Apply lighting models [S1].

**Formulas and Parameters:**
*   **Homothetic Distance:** The core metric for initial sampling. The specific formula is not detailed in the provided text, but it is used to define the "argument of the minimum" for heap initialization [S1].
*   **Field Evaluation Cost:** Computing field $f$ and gradient $\nabla f$ simultaneously costs $< 2 \times$ cost of $f$ alone [S1].
*   **Threshold:** No explicit convergence threshold $\epsilon$ is given in the notes, but standard ray marching applies [G].

**Pitfalls and Edge Cases:**
*   **Grazing Rays:** Less than $10^{-3}\%$ of rays may miss an isosurface crossing. These are exclusively grazing rays where the control polygons of the Hermite interpolant do not cross the isovalue 0 [S1].
*   **Heap Size:** The heap must be large enough to hold all overlapping primitives. If the skeleton is finely tessellated, the number of arguments to the homothetic distance minimum increases, requiring a larger heap or additional break conditions based on heap size [S1].
*   **Output Invariance:** Output invariance can be improved by combining the strategy with occluders generated from primitives considered in isolation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# SCALIS Rendering Algorithm
# Assumes:
# - Skeleton primitives are preprocessed into sphere-cone supports with AABBs.
# - HomotheticDistance(p, primitive) is defined.
# - EvaluateFieldAndGradient(p) returns (f, grad_f) efficiently.
# - QuadraticInterpolate(field_values) estimates root crossing.

def render_scalis(camera, primitives):
    # 1. Preprocessing (CPU)
    # Compute AABBs for all primitives based on sphere-cone supports
    for prim in primitives:
        prim.aabb = compute_aabb_from_sphere_cone_support(prim)
    
    # 2. Ray Generation (GPU)
    # Launch rays in 6 directions (±X, ±Y, ±Z)
    # Sample origins uniformly on the bounding box faces
    rays = generate_rays(camera, primitives.global_aabb, count=10000000) # For stats
    
    # 3. Ray Processing
    for ray in rays:
        # Initialize heap with primitives overlapping the initial homothetic interval
        heap = get_overlapping_primitives(ray, primitives)
        
        t = ray.origin_distance
        max_steps = 1000 # Implementation detail [G]
        
        for step in range(max_steps):
            # Evaluate field and gradient simultaneously
            # This costs < 2x single field evaluation
            field_val, grad = evaluate_field_and_gradient(ray.position(t), heap)
            
            # Check for surface hit
            if abs(field_val) < 1e-4: # Convergence threshold [G]
                hit_point = ray.position(t)
                normal = normalize(grad)
                return shade(hit_point, normal, ray)
            
            # Use quadratic polynomial interpolation for better step estimation
            # Requires previous field values or derivatives [S1]
            step_size = quadratic_step_estimate(field_val, grad, ray.direction)
            
            # Advance ray
            t += step_size
            
            # Update heap if new primitives become relevant
            # Based on homothetic distance minimum argument [S1]
            update_heap(heap, ray, t)
            
            if t > ray.max_distance:
                return background_color
                
    return background_color
```

## Subtopics

### Sphere Tracing
The parent method that SCALIS optimizes. Sphere tracing uses the SDF value directly as the step size, which is inefficient for grazing rays. [Distance-field ray marching](21-distance-field-ray-marching.md)

### Segment Tracing
A directional Lipschitz bound approach that can alleviate sphere tracing's grazing ray issues. SCALIS provides further reductions over this method. [Sphere Tracing and Step Rules](9643-sphere-tracing-and-step-rules.md)

## Operating characteristics
*   **Performance:** Reduces computation time by up to 70% compared to revised previous work [S1].
*   **Grazing Rays:** Specifically targets the "well-known problem" of maximal steps in sphere tracing along grazing rays, providing significant reduction in step count [S1].
*   **Memory:** Requires a heap large enough to store all primitives overlapping the initial homothetic distance intervals. Finely tessellated skeletons increase this requirement [S1].
*   **Accuracy:** Highly robust, with less than $10^{-3}\%$ of rays missing isosurface crossings, all of which are grazing rays due to Hermite data configurations [S1].
*   **Hardware:** Designed for efficient GPU implementation using an A-buffer [S1].

## Background
*   **Origin:** Introduced in the paper "Fast Ray Tracing of Scale‐Invariant Integral Surfaces" [S1].
*   **Context:** Integral surfaces are defined by skeletons and radius functions, allowing for interactive modeling of complex shapes. Previous methods suffered from slow rendering times, limiting interactive exploration of kernel parameters [S1].
*   **Development:** The method was developed to allow real-time exploration of a large range of skeletons and kernel parameters, enabling interactive modeling workflows [S1].

## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
