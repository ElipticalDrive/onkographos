# Indirect Diffuse Light and Ambient Occlusion Calculation

Also called: Indirect Diffuse Illumination, Indirect Light Calculation, Voxel Cone Tracing Indirect Diffuse Illumination

## Overview
This method calculates indirect diffuse lighting and ambient occlusion for real-time rendering by sampling the hemisphere above a surface point using a fixed set of symmetrically spaced cones [S1, unofficial]. It approximates the geometry term of the rendering equation by assigning higher weights to cones aligned with the surface normal, distinguishing it from random Monte Carlo sampling by using predetermined, fixed cone directions that ensure consistent hemisphere coverage [S1, unofficial].

## How it works
**Inputs:**
*   Surface position ($P$).
*   Surface normal ($N$).
*   Voxelized scene data containing indirect diffuse radiance.
*   Configuration parameters: Number of cones ($NUM\_CONES$), cone directions array ($coneDirections$), and cone weights array ($coneWeights$).

**Outputs:**
*   Indirect diffuse color ($Color$).
*   Ambient occlusion factor ($AO$).

**Data Structures:**
*   $coneDirections$: Array of $NUM\_CONES$ normalized 3D vectors. These are predetermined, fixed, and symmetrically spaced to cover the hemisphere $\Omega$ [S1, unofficial].
*   $coneWeights$: Array of $NUM\_CONES$ floating-point values. The sum of all weights equals 1.0 [S1, unofficial].
*   $coneTrace(\theta, P, D)$: A helper function that traces a cone with aperture $\theta$ starting at $P$ in direction $D$ against the voxel grid, returning the accumulated radiance and occlusion factor [S1, unofficial].

**Parameters:**
*   $NUM\_CONES$: Typically 6 [S1, unofficial].
*   Cone aperture ($\theta$): Determined by the cone configuration; cones are described as "large" to cover the hemisphere coarsely [S1, unofficial].

**Algorithm Steps:**
1.  Initialize $Color$ to $(0, 0, 0, 1)$ and $AO\_accum$ to $0.0$ [S1, unofficial].
2.  For each cone index $i$ from $0$ to $NUM\_CONES - 1$:
    a.  Calculate the cone direction $D_i = N \times coneDirections[i]$. This rotates the predefined symmetric direction into the local surface space defined by the normal [S1, unofficial].
    b.  Call $coneTrace(\theta, P, D_i)$ to retrieve the sampled radiance $R_i$ and the occlusion factor $O_i$ for this cone [S1, unofficial].
    c.  Update $Color$: $Color \leftarrow Color + coneWeights[i] \times R_i$ [S1, unofficial].
    d.  Update $AO\_accum$: $AO\_accum \leftarrow AO\_accum + coneWeights[i] \times O_i$ [S1, unofficial].
3.  Calculate final ambient occlusion: $AO = 1.0 - AO\_accum$ [S1, unofficial].
4.  Return the final indirect diffuse color: $FinalColor = Color \times AO$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
function INDIRECT_LIGHT_CALCULATION(position: vec3, normal: vec3, NUM_CONES: int, coneDirections: array[vec3], coneWeights: array[float], coneTrace: function) -> vec4:
    color = vec4(0.0)
    ao_accum = 0.0
    
    for i from 0 to NUM_CONES - 1:
        # Rotate the predefined cone direction into the local space of the surface normal
        # Note: The source notation "normal * coneDirections[i]" implies a rotation or basis transformation
        # from the canonical cone direction to the surface normal's local frame.
        cone_dir = rotate_vector_to_normal(coneDirections[i], normal)
        
        # Trace the cone against the voxelized scene
        # Returns a struct or pair containing radiance (vec3) and occlusion (float)
        result = coneTrace(cone_aperture, position, cone_dir)
        
        radiance = result.radiance
        occlusion = result.occlusion
        
        # Accumulate weighted radiance
        color = color + coneWeights[i] * radiance
        
        # Accumulate weighted occlusion
        ao_accum = ao_accum + coneWeights[i] * occlusion
        
    end for
    
    # Final ambient occlusion is the inverse of the accumulated occlusion
    ao = 1.0 - ao_accum
    
    # Apply ambient occlusion to the indirect diffuse color
    final_color = color * ao
    
    return final_color

# Helper to rotate a canonical direction vector to align with a surface normal
# This is a standard graphics operation not detailed in the source but required for implementation [G]
function rotate_vector_to_normal(canonical_dir: vec3, normal: vec3) -> vec3:
    # Construct a rotation matrix or quaternion that maps the canonical direction (e.g., (0,0,1)) to the normal
    # and apply it to canonical_dir.
    # A common approach is using a rotation matrix derived from the cross product and dot product
    # between the canonical up vector and the normal.
    pass
```

**Edge Cases and Pitfalls:**
*   **Geometry Term Approximation:** The method approximates the rendering equation's geometry term by weighting the normal-perpendicular cone more heavily. This is necessary because the cones have high apertures, making it difficult to pinpoint exact incidence angles [S1, unofficial].
*   **Weight Sum:** The total weights distributed among the cones must sum to 1.0 to ensure correct energy conservation in the approximation [S1, unofficial].
*   **Occlusion Calculation:** Ambient occlusion darkens corners and concavities by accumulating occlusion factors from all cones. If a cone is blocked, its contribution to the radiance is reduced, and its occlusion factor increases, leading to a darker final result [S1, unofficial].

## Operating characteristics
*   **Complexity:** Time complexity is linear with respect to the number of cones ($NUM\_CONES$) and the depth of the cone trace. Memory complexity is dominated by the voxel grid storage.
*   **Performance:** Using a fixed, small number of cones (e.g., 6) makes this method significantly faster than full Monte Carlo path tracing, suitable for real-time applications [S1, unofficial].
*   **Strengths:** Deterministic results due to fixed cone directions; efficient approximation of indirect lighting and ambient occlusion in a single pass [S1, unofficial].
*   **Limitations:** The coarse approximation with large cones may result in blurred indirect lighting and less accurate geometry term estimation compared to higher-sample methods [S1, unofficial].

## Background
This method is part of the Voxel Cone Tracing (VCT) pipeline, specifically addressing the calculation of indirect diffuse illumination and ambient occlusion. It was evaluated for real-time applications, building upon earlier work in voxelization and cone tracing for direct lighting [S1, unofficial]. The approach is reminiscent of Monte Carlo integration but replaces random sampling with a fixed, symmetric set of cones to ensure consistent hemisphere coverage [S1, unofficial].

## Sources
[S1] PDF Voxel Cone Tracing Evaluation for Real-Time Applications, https://kth.diva-portal.org/smash/get/diva2:1886204/FULLTEXT01.pdf, fetched 2026-10-27 (unofficial)
