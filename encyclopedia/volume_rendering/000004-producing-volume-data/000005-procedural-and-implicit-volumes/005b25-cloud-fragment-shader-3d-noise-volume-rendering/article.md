# Cloud Fragment Shader (3D Noise Volume Rendering)

Also called: Procedural Clouds via 3D Noise

## Overview
The Cloud Fragment Shader is a GPU-based technique for rendering volumetric clouds on a dome geometry by sampling a 3D noise volume texture [S1, unofficial]. It solves the problem of generating dynamic, drifting cloud formations with minimal CPU overhead by performing density calculation and color blending entirely within the fragment shader [S1, unofficial]. It differs from its parent [Procedural Noise Functions](1799-procedural-noise-functions.md) by operating on pre-computed 3D texture lookups rather than evaluating analytical noise functions per-fragment, and from its sibling [Alpha-Blended Cloud Rendering with Occlusion](20773-alpha-blended-cloud-rendering-with-occlusion.md) by lacking depth-aware occlusion mixing and multi-layer instancing, relying instead on simple alpha blending with the background sky [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `ModelPosition` (varying `vec3`): Scaled vertex position passed from the vertex shader.
    *   `Noise` (uniform `sampler3D`): 3D texture containing procedural noise data.
    *   `Offset` (uniform `vec3`): Dynamic offset for cloud drift and formation.
    *   `Cutoff` (uniform `float`): Threshold for cloud coverage.
    *   `Exponent` (uniform `float`): Controls dynamic range and edge sharpness.
    *   `CloudColor` (uniform `vec3`): RGB color of the cloud volume.
    *   `Bias` (uniform `float`): Intensity multiplier.
    *   `gl_Color.a` (built-in varying): Vertex alpha value.
*   **Output:**
    *   `gl_FragColor` (built-in output): Final fragment color with alpha.

**Data Structures:**
*   **3D Noise Texture:** A 3D texture array where the alpha channel (`a`) stores the noise value. The red, green, and blue channels are unused for density in this specific implementation [S1, unofficial].
*   **Uniforms:** Standard OpenGL uniform variables defined in the shader source [S1, unofficial].

**Steps:**
1.  **Receive Position:** The fragment shader receives the `ModelPosition` varying from the vertex shader. This position is already scaled by the `Scale` uniform to match the noise texture dimensions [S1, unofficial].
2.  **Sample Noise:** Sample the 3D noise texture using coordinates `ModelPosition + Offset`. Extract the alpha channel value [S1, unofficial].
    $$ n_{sampled} = \text{texture3D}(\text{Noise}, \text{ModelPosition} + \text{Offset}).a $$
3.  **Clamp Noise:** Apply a cutoff threshold. If the sampled noise is below `Cutoff`, set it to 0.0. Otherwise, subtract `Cutoff` to shift the range [S1, unofficial].
    $$ n_{clamp} = \begin{cases} 0.0 & \text{if } n_{sampled} < \text{Cutoff} \\ n_{sampled} - \text{Cutoff} & \text{otherwise} \end{cases} $$
4.  **Exponentiate and Bias:** Apply exponential mapping to control density falloff and edge sharpness, then multiply by `Bias` [S1, unofficial].
    $$ n_{out} = \text{Bias} \times (1.0 - \text{pow}(\text{Exponent}, n_{clamp})) $$
5.  **Compute Final Color:** Combine `CloudColor` with the processed noise alpha and the vertex alpha (`gl_Color.a`) [S1, unofficial].
    $$ \text{FinalColor} = \text{vec4}(\text{CloudColor}, n_{out} \times \text{gl\_Color.a}) $$
6.  **Output:** Write `FinalColor` to `gl_FragColor` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Main Function
def main():
    # 1. Sample 3D noise texture using scaled model position and dynamic offset
    # Note: texture3D is a standard GLSL function for 3D texture lookup
    noise_sample = texture3D(Noise, ModelPosition + Offset).a
    
    # 2. Clamp noise based on cutoff threshold
    if noise_sample < Cutoff:
        noise_clamped = 0.0
    else:
        noise_clamped = noise_sample - Cutoff
        
    # 3. Apply exponential density mapping and bias
    # Formula: n_out = Bias * (1.0 - Exponent^n_clamped)
    noise_final = Bias * (1.0 - pow(Exponent, noise_clamped))
    
    # 4. Compute final color
    # Combine CloudColor with noise alpha and vertex alpha
    # Note: gl_Color.a is the vertex alpha passed through the pipeline
    final_alpha = noise_final * gl_Color.a
    final_color = vec4(CloudColor, final_alpha)
    
    # 5. Output fragment color
    gl_FragColor = final_color
```

**Parameters:**
*   `Scale` (uniform float): Scales the incoming vertex position relative to the noise texture size. Allows finer control of cloud size [S1, unofficial].
*   `Offset` (uniform vec3): Adds a per-frame offset to texture coordinates. The x-component simulates cloud drift, and the y-component controls formation [S1, unofficial].
*   `Cutoff` (uniform float): Threshold percentage for cloud coverage. Values below this are clamped to zero alpha [S1, unofficial].
*   `Exponent` (uniform float): Controls the dynamic range of the noise after clamping. Higher values sharpen edges and reduce density [S1, unofficial].
*   `CloudColor` (uniform vec3): RGB color of the cloud volume [S1, unofficial].
*   `Bias` (uniform float): Multiplier applied after exponentiation to adjust the final alpha/color intensity [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Vertex Alpha Ignored:** The fixed-function `gl_FrontColor` is ignored when shaders are enabled. The vertex alpha must be preserved via `gl_Color.a` and explicitly multiplied with the computed noise alpha in the final output [S1, unofficial].
*   **Blending:** The final cloud color is blended with the background sky using `SRC_ALPHA` and `ONE_MINUS_SRC_ALPHA` factors [S1, unofficial].

## Subtopics

### [Alpha-Blended Cloud Rendering with Occlusion](20773-alpha-blended-cloud-rendering-with-occlusion.md)
A fragment-based cloud rendering technique that uses 3D noise sampling with alpha clipping to define volume boundaries, combined with a multi-term lighting model (diffuse, subsurface scattering approximation, view brightening, and shadows) for surface shading. It differs from its parent [Cloud Fragment Shader (3D Noise Volume Rendering)](16861-cloud-fragment-shader-3d-noise-volume-rendering.md) by replacing simple background blending with a depth-aware occlusion mixing system that allows clouds to interact with opaque scene geometry, and by using instanced rendering to stack multiple cloud layers with varying density thresholds.

## Operating characteristics
*   **Time Complexity:** O(1) per fragment for the noise lookup and post-processing steps, assuming constant-time texture sampling [S1, unofficial].
*   **Memory Complexity:** O(N) for the 3D noise texture, where N is the volume resolution [S1, unofficial].
*   **Performance:** The method is efficient for real-time rendering as it offloads density calculation to the GPU fragment shader [S1, unofficial].
*   **Strengths:** Simple implementation, dynamic cloud movement via offset uniforms, and low CPU overhead [S1, unofficial].
*   **Limitations:** Lacks depth-aware occlusion, multi-layer interaction, and complex lighting models found in more advanced techniques [S1, unofficial].
*   **When to Use:** Suitable for simple sky dome cloud rendering where performance is critical and complex volumetric interactions are not required [S1, unofficial].
*   **When Not to Use:** When realistic volumetric lighting, occlusion against terrain, or multi-layer cloud systems are needed [S1, unofficial].

## Background
The Cloud Fragment Shader was implemented in the OpenSceneGraph (OSG) library for the `CloudDome` class to render volumetric clouds on a dome geometry [S1, unofficial]. It utilizes GLSL (OpenGL Shading Language) and requires specific extensions (`GL_ARB_fragment_shader`, `GL_ARB_vertex_shader`, `GL_ARB_shader_objects`, `GL_ARB_shading_language_100`) to be supported [S1, unofficial]. The technique was described by George Tarantilis in a thesis on simulating clouds with procedural texturing techniques using the GPU [S1, unofficial].

## Sources
[S1] DTIC ADA427539: Simulating Clouds with Procedural Texturing Techniques Using the GPU, https://archive.org/details/DTIC_ADA427539, fetched 2026-10-26 (unofficial)
