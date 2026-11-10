# Phong Shading with Shadows

Also called: Phong Relighting, Standard Illumination Model

## Overview
Phong Shading with Shadows computes the final pixel color for implicit surfaces by applying the Phong reflection model modulated by shadow factors [S3, unofficial]. It extends [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by adding specular highlights and shadow casting, and evaluates the shading model per sample during ray marching, unlike [Sphere Tracing with Shading](3812-sphere-tracing-with-shading.md) which may decouple shading logic from traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A signed distance field function $SDF(p)$, a camera position $C_{cam}$, a light source definition (position $L_{pos}$ or direction $L_{dir}$, color $I$, intensity), and material parameters.
*   **Output:** A final color $C$ for each pixel.

**Data Structures:**
The material properties are stored in a `material` struct [S3, unofficial]:
*   `vec3 ka`: Ambient color/intensity.
*   `float kd`: Diffuse reflection coefficient.
*   `float ks`: Specular reflection coefficient.
*   `float kp`: Shininess exponent.

The light source is defined by a `light` struct [S3, unofficial]:
*   `vec3 c`: Light color.
*   `float intensiv`: Light intensity.

**Steps:**
1.  **Ray Marching:** Perform sphere tracing or ray marching to find the intersection point $p$ on the SDF surface.
2.  **Surface Normal Recovery:** Calculate the surface normal $n$ at point $p$ by normalizing the gradient of the SDF [S1, unofficial]:
    $$ n = \frac{\nabla SDF(p)}{\|\nabla SDF(p)\|} $$
3.  **Light Direction Calculation:** Determine the normalized light direction vector $l$. For a point light at position $L_{pos}$:
    $$ l = \frac{L_{pos} - p}{\|L_{pos} - p\|} $$
    [S3, unofficial] For a directional light, use the predefined direction vector.
4.  **Shadow Factor Calculation:** Compute the shadow factor $sh$ by casting a shadow ray from $p$ towards the light source [S3, unofficial].
5.  **Phong Reflection Model:** Calculate the diffuse and specular components [S3, unofficial]:
    *   **Diffuse:**
        $$ C_{diff} = k_d I \max(n \cdot l, 0) $$
    *   **Specular:**
        Calculate the reflection vector $r$:
        $$ r = \text{reflect}(-v, n) $$
        where $v = \text{normalize}(C_{cam} - p)$ is the normalized view direction.
        $$ C_{spec} = k_s \max(r \cdot l, 0)^{k_p} $$
6.  **Final Color:** Combine ambient, diffuse, and specular contributions, modulated by the shadow factor [S3, unofficial]:
    $$ C = (k_a + C_{diff} + C_{spec}) \times sh $$

**Parameters:**
*   $k_a$: Ambient constant.
*   $k_d$: Diffuse constant.
*   $k_s$: Specular constant.
*   $k_p$: Shininess constant. Higher values result in smoother, shinier specular highlights [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Noise:** If the SDF is noisy, the gradient $\nabla SDF$ will be noisy, causing flickering normals [S1, unofficial].
*   **Shadow Acne:** Casting shadow rays from $p$ can cause self-intersection artifacts. A small epsilon offset $\epsilon$ along $n$ is necessary to start the shadow ray slightly off the surface [G].
*   **View Direction:** For volume rendering, the view direction $v$ is computed as $C_{cam} - p$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reflect(I, N):
    """Computes reflection vector: R = 2*dot(N,I)*N - I"""
    return 2 * dot(N, I) * N - I

def phong_shading_with_shadows(sdf, p, light_pos, light_color, light_intensity, 
                               mat_ka, mat_kd, mat_ks, mat_kp, cam_pos, 
                               shadow_mode='hard', shadow_epsilon=0.001):
    """
    Computes Phong shading with shadows for a point p on an SDF surface.
    """
    # 1. Surface Normal Recovery
    eps = 0.001
    grad_x = (sdf(p + vec3(eps, 0, 0)) - sdf(p - vec3(eps, 0, 0))) / (2 * eps)
    grad_y = (sdf(p + vec3(0, eps, 0)) - sdf(p - vec3(0, eps, 0))) / (2 * eps)
    grad_z = (sdf(p + vec3(0, 0, eps)) - sdf(p - vec3(0, 0, eps))) / (2 * eps)
    gradient = vec3(grad_x, grad_y, grad_z)
    n = normalize(gradient)
    
    # 2. Directions
    l = normalize(light_pos - p)
    v = normalize(cam_pos - p)
    
    # 3. Shadow Factor
    sh = 1.0
    if shadow_mode == 'hard':
        t = shadow_epsilon
        max_dist = distance(light_pos, p)
        while t < max_dist:
            if sdf(p + n * shadow_epsilon + l * t) < 0:
                sh = 0.0
                break
            t += shadow_epsilon
    else:
        # Soft shadow trace requires penumbra width parameter [G]
        sh = 1.0 
            
    # 4. Phong Components
    diff_term = max(dot(n, l), 0.0)
    C_diff = mat_kd * light_color * light_intensity * diff_term
    
    r = reflect(-v, n)
    spec_term = max(dot(r, l), 0.0)
    C_spec = mat_ks * light_color * light_intensity * (spec_term ** mat_kp)
    
    # 5. Final Color
    C = (mat_ka + C_diff + C_spec) * sh
    return C
```

## Subtopics

### [Lighting Transfer Function](6468-lighting-transfer-function.md)
The Lighting Transfer Function (LTF) is a 2D lookup table technique that integrates multiple illumination models into a single, constant-cost evaluation for volume rendering. It solves the problem of efficiently applying complex illustrative shading (such as contour enhancement, cartoon shading, or metal shading) by storing ambient, diffuse, specular, and opacity enhancement components indexed by the dot products of the normalized gradient with the light and half-way vectors. Unlike standard Phong shading which calculates lighting per sample, the LTF decouples the shading model from the ray marching loop, allowing different visual styles to be swapped without changing the rendering algorithm.

### [Differentiable Blinn-Phong Light Estimation](8621-differentiable-blinn-phong-light-estimation.md)
This method solves the inverse problem of estimating the position and intensity of point light sources given an observed image, known scene geometry, and material properties. It formulates light estimation as an optimization problem that minimizes the photometric error between a rendered image and the target image using a differentiable rendering pipeline based on the Blinn-Phong model. It distinguishes itself from sibling methods like [Lighting Transfer Function](6468-lighting-transfer-function.md) by explicitly modeling 3D point light sources and handling non-distant lighting parallax, whereas alternatives like Spherical Harmonics are limited to low-frequency or distant lighting approximations.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel for ray marching, plus additional steps for shadow tracing [G].
*   **Memory Complexity:** $O(1)$ per pixel for shading calculations, plus storage for the SDF representation [G].
*   **Strengths:** Provides specular highlights and shadows that enhance the perception of depth and material properties [S3, unofficial]. Simple to implement and understand.
*   **Limitations:** Computationally expensive if soft shadows are used extensively. Sensitive to SDF gradient quality. Does not account for global illumination effects like color bleeding [G].
*   **When to Use:** When realistic specular highlights and shadows are desired for implicit surfaces or neural fields [S1, unofficial].
*   **When Not to Use:** When performance is critical and soft shadows are not required, or when global illumination is needed [G].

## Background
The method applies the Phong reflection model to implicit surfaces, computing per-pixel normals via SDF gradients to evaluate lighting [S1, unofficial]. It is used in volume rendering and SDF-based rendering to provide shading for surfaces defined implicitly [S3, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] MTCut: GPU-based Marching Tetra Cuts, https://doi.org/10.2312/localchapterevents/tpcg/tpcg07/037-044, fetched 2026-10-26 (unofficial)
[S3] trimesh33/sdf, https://github.com/trimesh33/sdf, fetched 2026-10-27 (unofficial)
