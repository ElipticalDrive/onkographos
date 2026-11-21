# Inter-reflection Modeling

Also called: Flashlight Inter-reflection, Secondary Intersection Point Ray Marching

## Overview
Inter-reflection Modeling is a technique for approximating indirect illumination in inverse rendering by tracing a single ray from a surface point in its reflective direction to a secondary intersection point, computing the radiance at that point, and blending it into the primary point's output [S1]. Unlike methods such as IRON and WildLight that assume collocated camera and light sources, this approach accommodates variations in both incident and outgoing light directions, making it suitable for non-collocated capture setups [S1].

## How it works
**Inputs and Outputs**
*   **Inputs**: Surface point $x$, surface normal $n$ at $x$, view direction $\omega_i$, light source position, Signed Distance Field (SDF) of the scene, and a neural network (MLP) that predicts a blending coefficient $\gamma$ based on roughness.
*   **Outputs**: Indirect illumination component $f_{ir}$ added to the shading model.

**Data Structures**
*   **SDF**: A function $SDF(p)$ returning the distance to the nearest surface. Negative values indicate interior/occlusion [S1].
*   **Blending Coefficient Network**: An MLP taking roughness as input and outputting a scalar $\gamma \in [0, 1]$ [S1].

**Steps**
1.  **Compute Reflective Direction**: Given the view direction $\omega_i$ and surface normal $n$, calculate the reflective direction $\omega_r$ [S1]:
    $$ \omega_r = 2 \times (n \cdot \omega_i) n - \omega_i $$
    *Note: The source formula $wr = 2 \times n - wi$ appears to be a simplified notation or typo for the standard reflection vector formula involving the dot product. The standard vector reflection is used here for correctness [G].*

2.  **Trace Secondary Intersection**: Cast a ray from $x$ in direction $\omega_r$ to find the secondary intersection point $x'$ [S1]. This is typically done via ray marching or SDF traversal.

3.  **Occlusion Check**: Determine if $x'$ is occluded from the light source [S1]:
    *   Sample 20 points uniformly along the path between the light source and $x'$ [S1].
    *   Evaluate the SDF at each sampled point.
    *   If any sampled point has an SDF value $< 0$, the light is considered occluded ($o_{x'} = 0$) [S1].
    *   Otherwise, the light is unobstructed ($o_{x'} = 1$) [S1].

4.  **Compute Outgoing Radiance**: If $x'$ is unobstructed, compute the outgoing radiance at $x'$ using the flashlight’s incoming radiance $L_{ind}$ [S1]. The source references a BRDF implementation (roughplastic model) for this calculation [S1].

5.  **Compute Blending Coefficient**: Input the roughness property of the 3D point into the MLP to obtain the blending coefficient $\gamma$ [S1].

6.  **Combine Indirect Illumination**: Calculate the indirect illumination component $f_{ir}$ by combining the outgoing radiance at $x'$ with the blending coefficient $\gamma$ and the occlusion term [S1]. The exact formula is not explicitly provided in the text, but Figure 10(d) implies a structure involving $\gamma$, $L_{ind}$, and potentially geometric terms like distance or normal alignment [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_interreflection(x, n, wi, light_pos, sdf, roughness_mlp):
    """
    Computes the indirect illumination component f_ir for point x.
    
    Args:
        x: Point position (vec3)
        n: Surface normal at x (vec3, normalized)
        wi: View direction (vec3, normalized)
        light_pos: Light source position (vec3)
        sdf: Function returning SDF value for a point
        roughness_mlp: Neural network returning blending coefficient gamma from roughness
        
    Returns:
        f_ir: Indirect illumination contribution (scalar or vec3)
    """
    # Step 1: Compute reflective direction
    # Standard reflection formula: wr = 2 * dot(n, wi) * n - wi
    # Note: Source formula "wr = 2 * n - wi" is likely a simplified notation.
    # Using standard vector reflection for correctness [G].
    dot_val = dot(n, wi)
    wr = 2.0 * dot_val * n - wi
    
    # Step 2: Trace secondary intersection point x'
    # Implementation detail: Ray march from x along wr to find x'
    # This requires a ray-marching function for SDFs.
    x_prime = ray_march_sdf(x, wr, sdf)
    
    # Step 3: Occlusion check
    # Sample 20 points between light_pos and x_prime
    num_samples = 20
    occluded = False
    for i in range(num_samples):
        t = i / (num_samples - 1)
        sample_point = lerp(light_pos, x_prime, t)
        if sdf(sample_point) < 0:
            occluded = True
            break
            
    if occluded:
        return 0.0
        
    # Step 4: Compute outgoing radiance at x'
    # Using the flashlight's incoming radiance L_ind
    # The source mentions a roughplastic BRDF model [S1].
    # We assume L_ind is available or computed from light properties.
    # For simplicity, we assume L_ind is the radiance arriving at x' from the light.
    # In a full renderer, this involves evaluating the BRDF at x'.
    L_ind = compute_light_radiance(light_pos, x_prime, sdf)
    
    # Step 5: Compute blending coefficient gamma
    # Get roughness from the material network at x (or x'?)
    # Source says "conditioned on the roughness property of the 3D point"
    # and "detaching the roughness of the secondary intersection point x' prior to its input"
    # This implies we use the roughness of x, but detach gradients from x' roughness if it were used.
    # Assuming roughness is a property of x.
    roughness = get_roughness(x)
    gamma = roughness_mlp(roughness)
    
    # Step 6: Combine
    # Formula not explicitly given, but Figure 10(d) suggests f_ir depends on gamma, L_ind, and occlusion.
    # A common form is f_ir = gamma * L_ind * occlusion_factor.
    # Since occlusion is binary here, we use 1 if not occluded.
    f_ir = gamma * L_ind
    
    return f_ir
```

**Parameters**
*   **Occlusion Samples**: 20 points uniformly sampled between the light source and $x'$ [S1].
*   **Blending Coefficient $\gamma$**: Output of an MLP conditioned on roughness [S1].
*   **BRDF Model**: Roughplastic model with GGX distribution parameter [S1].

**Edge Cases and Pitfalls**
*   **Optimization Ambiguity**: Detaching the roughness of $x'$ from the gradient flow into the blending coefficient network prevents correlation-induced ambiguity, leading to more precise material decomposition [S1].
*   **Geometric Reconstruction**: Disabling the optimization of local geometry at $x'$ reduces optimization complexity and improves geometric reconstruction, especially in concave regions [S1].

## Operating characteristics
*   **Time Complexity**: Depends on the cost of ray marching to find $x'$ and the 20 SDF evaluations for occlusion. The MLP evaluation is negligible.
*   **Memory Complexity**: Low, requiring storage for the SDF and the MLP weights.
*   **Performance**: Training requires approximately 9 hours on a single RTX 3090 GPU with 24GB of memory [S1].
*   **Strengths**: Handles non-collocated camera and light setups, unlike IRON and WildLight [S1]. Improves geometric reconstruction in concave areas by disabling local geometry optimization at $x'$ [S1].
*   **Limitations**: Approximates indirect illumination with a single bounce. Accuracy depends on the quality of the SDF and the MLP.

## Background
Inter-reflection Modeling was introduced in the context of Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization (PIR) [S1]. It builds upon previous methods like IRON and WildLight, which used simplified BRDF models and assumed collocated camera and light sources [S1]. The method was developed to address deviations in capture setups where the camera and light are not collocated [S1].

## Sources
[S1] PIR: Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization, https://doi.org/10.1109/3dv66043.2025.00055, fetched 2026-10-26
