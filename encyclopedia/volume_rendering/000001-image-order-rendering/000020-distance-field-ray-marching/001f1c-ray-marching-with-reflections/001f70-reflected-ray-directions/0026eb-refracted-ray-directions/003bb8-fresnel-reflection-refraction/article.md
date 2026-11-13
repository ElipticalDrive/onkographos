# Fresnel Reflection/Refraction

Also called: Fresnel Reflection

## Overview
Calculates the Fresnel reflectance for unpolarized light at an interface between two media with different indices of refraction [S1, unofficial]. It determines the ratio of reflected to refracted light energy, which is essential for physically based rendering of dielectrics and metals. It sets itself apart from sibling methods by providing the weighting factor ($k_R$) used to blend reflected and refracted ray contributions, rather than calculating the ray directions themselves [S1, unofficial].

## How it works
**Inputs:**
*   `incom`: Incoming light direction or view direction (vector).
*   `normal`: Surface normal at the intersection point (vector).
*   `index_internal`: Refractive index of the internal medium (scalar).
*   `index_external`: Refractive index of the external medium (scalar).

**Outputs:**
*   `reflectance`: A scalar value between 0.0 and 1.0 representing the fraction of light reflected [S1, unofficial].

**Data Structures:**
No complex data structures are required. The method operates on standard vector and scalar types.

**Steps:**
1.  Calculate the ratio of indices of refraction: $\eta = \frac{\text{index\_internal}}{\text{index\_external}}$ [S1, unofficial].
2.  Calculate the cosine of the angle of incidence: $\cos\theta_1 = \text{dot}(\text{incom}, \text{normal})$ [S1, unofficial].
3.  Calculate the squared cosine of the angle of refraction: $\cos^2\theta_2 = 1.0 - (\eta \cdot \eta) \cdot (1.0 - \cos\theta_1 \cdot \cos\theta_1)$ [S1, unofficial].
4.  Check for total internal reflection:
    *   If $\cos^2\theta_2 < 0.0$, return $1.0$ [S1, unofficial].
    *   Otherwise, calculate $\cos\theta_2 = \sqrt{\cos^2\theta_2}$ [S1, unofficial].
5.  Calculate the Fresnel s-polarized reflectance term:
    $$ \text{fresnel\_rs} = \frac{\text{index\_internal} \cdot \cos\theta_1 - \text{index\_external} \cdot \cos\theta_2}{\text{index\_internal} \cdot \cos\theta_1 + \text{index\_external} \cdot \cos\theta_2} $$
    [S1, unofficial].
6.  Calculate the Fresnel p-polarized reflectance term:
    $$ \text{fresnel\_rp} = \frac{\text{index\_internal} \cdot \cos\theta_2 - \text{index\_external} \cdot \cos\theta_1}{\text{index\_internal} \cdot \cos\theta_2 + \text{index\_external} \cdot \cos\theta_1} $$
    [S1, unofficial].
7.  Return the average reflectance for unpolarized light:
    $$ \text{reflectance} = (\text{fresnel\_rs}^2 + \text{fresnel\_rp}^2) \cdot 0.5 $$
    [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fresnel_reflection(incom, normal, index_internal, index_external):
    # Step 1: Calculate eta
    eta = index_internal / index_external
    
    # Step 2: Calculate cos_theta1
    cos_theta1 = dot(incom, normal)
    
    # Step 3: Calculate cos_theta2 squared
    cos_theta2_sq = 1.0 - (eta * eta) * (1.0 - cos_theta1 * cos_theta1)
    
    # Step 4: Handle total internal reflection
    if cos_theta2_sq < 0.0:
        return 1.0
    
    # Step 5: Calculate cos_theta2
    cos_theta2 = sqrt(cos_theta2_sq)
    
    # Step 6: Calculate Fresnel s-polarized reflectance
    fresnel_rs = (index_internal * cos_theta1 - index_external * cos_theta2) / \
                 (index_internal * cos_theta1 + index_external * cos_theta2)
    
    # Step 7: Calculate Fresnel p-polarized reflectance
    fresnel_rp = (index_internal * cos_theta2 - index_external * cos_theta1) / \
                 (index_internal * cos_theta2 + index_external * cos_theta1)
    
    # Step 8: Return average reflectance
    return (fresnel_rs * fresnel_rs + fresnel_rp * fresnel_rp) * 0.5
```

**Usage in Shading:**
In a fragment shader, the Fresnel factor $k_R$ is calculated using the view direction (often negated eye vector) and the surface normal [S1, unofficial]. The transmission factor is $k_T = 1.0 - k_R$ [S1, unofficial]. The final color is a linear combination of the reflected color and the refracted color:
$$ \text{color} = k_R \cdot \text{reflectColor} + k_T \cdot \text{refractColor} $$
[S1, unofficial]. The reflected ray direction is calculated as $\text{reflectRay} = \text{normalize}(\text{reflect}(\text{eye}, \text{normal}))$ and the refracted ray direction as $\text{refractRay} = \text{normalize}(\text{refract}(\text{eye}, \text{normal}, 1. / \text{uniforms.IOR}))$ [S1, unofficial].

## Subtopics

### Complex-Indexed Fresnel Tint
An OpenPBR-style edge coloring technique for metals using complex-indexed Fresnel calculations, allowing for non-white edge highlights [S2, unofficial]. It uses an `edgeTint` parameter to control the color of the edge tint; a white value reproduces the Schlick approximation exactly [S2, unofficial]. A key pitfall is that standard real-index Fresnel must not be used; complex-indexed Fresnel calculations are required to correctly model metallic edge colors [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The calculation involves basic arithmetic operations (dot product, square root, division, multiplication) and is $O(1)$ in time and memory [S1, unofficial].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
The standard Fresnel equations provide physically accurate results for dielectrics [S1, unofficial]. However, they assume unpolarized light and do not account for thin-film interference or metallic absorption without modification [S2, unofficial].

**When to use it and when not:**
Use it for rendering dielectric interfaces and as a base for metallic reflections [S1, unofficial]. Do not use the standard real-index version for metals if accurate edge tinting is required; use Complex-Indexed Fresnel Tint instead [S2, unofficial].

## Background
The method is implemented in WebGPU-based voxel cone tracing and photon engine projects [S1, S2, unofficial]. The complex-indexed variant is part of advanced material features in the photon engine, referencing OpenPBR standards [S2, unofficial].

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
