# Diffuse Cone Distribution

Also called: 5-cone diffuse distribution, normal-perpendicular cone set

## Overview
Diffuse Cone Distribution is a specific sampling strategy for approximating indirect diffuse lighting in Voxel Cone Tracing. It solves the problem of efficiently estimating hemispherical irradiance using a minimal number of cone traces. It sets itself apart from its siblings by using a fixed, deterministic geometric arrangement of five cones rather than stochastic rotation or multiple bounces, trading visual smoothness for computational speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Surface normal vector $\mathbf{n}$, surface position $\mathbf{p}$.
- Output: Accumulated diffuse radiance $L_{diffuse}$.

**Data Structures:**
- Cone: Defined by origin $\mathbf{o}$, direction $\mathbf{d}$, and aperture angle $\alpha$.
- Voxel Tracer: Function `trace_cone(o, d, alpha)` returning accumulated color and occlusion.

**Parameters:**
- Number of cones: 5 [S1, unofficial].
- Cone aperture: $30^\circ$ for all cones [S1, unofficial].
- Offset distance: Small epsilon $\epsilon$ to prevent self-intersection [S1, unofficial].

**Steps:**
1. **Generate Cone Directions:** Create five direction vectors relative to the surface normal $\mathbf{n}$:
   - Cone 0: $\mathbf{d}_0 = \mathbf{n}$
   - Cones 1-4: Four vectors $\mathbf{d}_1, \mathbf{d}_2, \mathbf{d}_3, \mathbf{d}_4$ lying in the plane perpendicular to $\mathbf{n}$. These are typically spaced $90^\circ$ apart around the normal (e.g., along local tangent and bitangent axes).
2. **Set Origins:** For each cone $i$, set the origin $\mathbf{o}_i = \mathbf{p} + \epsilon \mathbf{d}_i$. The offset $\epsilon$ is critical to avoid sampling the surface voxel itself, which causes self-shadowing and self-lighting artifacts [S1, unofficial].
3. **Trace Cones:** For each cone $i$, perform a voxel cone trace using direction $\mathbf{d}_i$, aperture $30^\circ$, and origin $\mathbf{o}_i$.
4. **Accumulate:** Sum the radiance contributions from all five traces.
   $$ L_{diffuse} = \sum_{i=0}^{4} L_i $$
   where $L_i$ is the result of the $i$-th cone trace.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def diffuse_cone_distribution(surface_normal, surface_position, epsilon=0.001):
    # Define the 5 cone directions
    # Cone 0 is along the normal
    directions = [surface_normal]
    
    # Cones 1-4 are perpendicular to the normal
    # Assuming a local coordinate system where tangent and bitangent are known
    # or derived via Gram-Schmidt if not provided
    tangent, bitangent = compute_orthonormal_basis(surface_normal)
    
    # Four perpendicular directions at 90 degree intervals
    directions.append(tangent)
    directions.append(bitangent)
    directions.append(-tangent)
    directions.append(-bitangent)
    
    accumulated_light = vec3(0.0)
    aperture = 30.0 # degrees
    
    for direction in directions:
        # Offset origin to avoid self-intersection artifacts
        origin = surface_position + direction * epsilon
        
        # Trace the cone in the voxel grid
        # Returns accumulated radiance for this cone
        light = trace_voxel_cone(origin, direction, aperture)
        
        accumulated_light += light
        
    return accumulated_light
```

**Edge Cases:**
- **Self-Shadowing:** If $\epsilon$ is too small, the cone origin may intersect the surface geometry, causing incorrect occlusion or self-lighting [S1, unofficial].
- **Blocky Artifacts:** With only 5 cones, the lighting approximation is coarse, leading to visible blockiness in smooth gradients [S1, unofficial].

## Subtopics

### [Random Cone Rotation](5911-random-cone-rotation.md)
Mitigates blocky artifacts by rotating the cone orientations randomly around the surface normal. It trades visual coherence for stochastic noise, unlike the deterministic 5-cone set.

### [Two-Bounce Indirect Illumination](4558-two-bounce-indirect-illumination.md)
Extends the approximation to include secondary bounces, offering higher accuracy at greater cost.

### [Multi-Cone Shading](5176-multi-cone-shading.md)
Uses a larger number of cones for smoother results, increasing computational load.

## Operating characteristics
- **Time Complexity:** $O(1)$ cone traces per surface point, specifically 5 traces. Each trace is $O(\log N)$ or $O(N)$ depending on the voxel traversal implementation, but the constant factor is very low compared to full path tracing.
- **Memory Complexity:** Minimal additional memory; relies on the existing voxel grid.
- **Strengths:** Extremely fast; suitable for real-time applications where performance is critical. Simple to implement.
- **Limitations:** Low quality due to coarse sampling; visible blockiness and noise (if not randomized); inaccurate for complex indirect lighting scenarios.
- **When to use:** Real-time rendering with strict performance budgets, simple scenes, or as a base layer for further refinement.
- **When not to use:** High-fidelity offline rendering, scenes with complex indirect lighting, or when visual smoothness is prioritized over speed.

## Background
This distribution was proposed in an unofficial report on Voxel Cone Tracing implementation details [S1, unofficial]. It represents a pragmatic compromise in the evolution of real-time global illumination, moving from simple hemisphere sampling to more structured cone-based approximations. The specific choice of 5 cones and $30^\circ$ aperture was empirically determined to provide "good quality" for the computational cost in that context.

## Sources
[S1] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
