# Boundary Enhancement


## Overview
Boundary enhancement is a compositing technique designed to improve the visibility of object boundaries and material interfaces within volume data [S1][S2, unofficial]. It addresses the problem of indistinct edges in standard volume rendering by emphasizing transitions between different tissues or structures, often used in conjunction with silhouette techniques in medical illustration [S2, unofficial]. Unlike standard direct volume rendering which relies on opacity and color accumulation, this method modifies the shading or opacity contribution based on boundary detection [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sequence of volume samples $(c_i, \alpha_i)$ along a ray, where $c_i$ is color and $\alpha_i$ is opacity [G]. For bump mapping variants, inputs also include classified intensity values $i_{class}$ and a bump strength parameter $v_{bump}$ [S3].
- Output: Enhanced pixel color $C_{final}$ with emphasized boundaries [S1][S2, unofficial].

**Data Structures:**
- Sample buffer: Stores $(c_i, \alpha_i, i_{class})$ for each ray step [S3].
- Gradient vector: Computed on-the-fly from intensity values to determine surface normals [S3].

**Steps:**
1. **Initialization:** Initialize accumulators $C_{acc} = \vec{0}$ and $\alpha_{acc} = 0$ [G].
2. **Ray Loop:** Iterate through samples front-to-back. For each sample, compute transmittance $T = 1 - \alpha_{acc}$ [G].
3. **Boundary Enhancement:**
   - Compute modified intensity for illumination: $i_{sample} = (i_{class} \cdot v_{bump}) - (v_{bump} - 1)$ [S3].
   - Modulate the color contribution using the modified intensity to enhance boundary illumination: $c_{enhanced} = c_i \cdot \frac{i_{sample}}{\max(i_{class}, \epsilon)}$ [G] (where $\epsilon$ prevents division by zero).
   - Update accumulators: $C_{acc} \leftarrow C_{acc} + T \cdot \alpha_i \cdot c_{enhanced}$ and $\alpha_{acc} \leftarrow \alpha_{acc} + T \cdot \alpha_i$ [G].
4. **Background Composite:** Final color $C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}$ [G].

**Formulas:**
- Standard compositing: As defined in parent [G].
- Bump mapping intensity modification [S3]:
  $$
  i_{sample\_j} = (i_{class\_j} \cdot v_{bump}) - (v_{bump} - 1)
  $$
  Where:
  - $i_{sample\_j}$: Sample intensity for illumination computation.
  - $i_{class\_j}$: Classified sample intensity.
  - $v_{bump}$: Bump value (strength of enhancement).

**Parameters:**
- $v_{bump}$: Strength of the bump effect. Typical value: 1.5 [S3]. If near zero, no enhancement is visible [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def boundary_enhancement_compositing(samples, background_color, v_bump=1.5):
    """
    Boundary enhancement compositing loop.
    samples: list of (color: vec3, opacity: float, classified_intensity: float)
    background_color: vec3
    v_bump: float, strength of bump mapping enhancement
    """
    C_acc = vec3(0.0)
    alpha_acc = 0.0
    epsilon = 1e-6
    
    for c_i, alpha_i, i_class in samples:
        transmittance = 1.0 - alpha_acc
        
        # Boundary Enhancement Logic
        if v_bump > 0.0 and i_class > epsilon:
            # Perturb intensity for illumination at boundaries
            i_sample = (i_class * v_bump) - (v_bump - 1.0)
            # Modulate color contribution using modified intensity
            c_enhanced = c_i * (i_sample / i_class)
        else:
            c_enhanced = c_i
            
        # Accumulate
        C_acc = C_acc + transmittance * alpha_i * c_enhanced
        alpha_acc = alpha_acc + transmittance * alpha_i
        
        # Early ray termination
        if alpha_acc >= 1.0:
            break
            
    return C_acc + (1.0 - alpha_acc) * background_color
```

## Subtopics

### [Temporal Gradient Enhancement](8138-temporal-gradient-enhancement.md)
Temporal Gradient Enhancement is a boundary enhancement technique for time-varying volume data that highlights features varying rapidly over time by computing gradients in the time dimension and modifying shading/opacity based on these gradients. Unlike standard animation which renders individual time steps independently, this method uses data from previous and next time steps for the current rendering to maintain coherence. It sets itself apart from spatial enhancement techniques (such as those using Non-Photorealistic Rendering strokes) by avoiding artifacts introduced into animations, ensuring visual coherence from one time step to the next.

### [Feature Enhancement](12204-feature-enhancement.md)
Feature Enhancement is a compositing technique that adds texture and contrast to a volume render, furthering the effects of edge enhancement by increasing the visibility of fine details. In the context of 3D Gaussian Splatting, it refers to a lightweight network that refines coarse RGB outputs by decoding feature vectors assigned to each Gaussian, compensating for missing high-frequency details. It builds on Boundary Enhancement by addressing texture fidelity rather than just boundary sharpness.

### Boundary Enhancement via Bump Mapping
Boundary Enhancement via Bump Mapping is a technique to enhance anatomical structures clipped by cutting planes by perturbing surface normals (gradients) to simulate bump mapping, improving the perception of underlying structures [S3]. It applies bump mapping to cutting surfaces during volume rendering, computing gradients on-the-fly using intensity values after classification [S3]. It manipulates the gradient direction at the boundary of structures while keeping other gradients unchanged, causing illumination changes at boundaries [S3].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples per ray, plus overhead for on-the-fly gradient computation and intensity modification [S3].
- **Memory Complexity:** Low, as gradients and modified intensities are computed on-the-fly [S3].
- **Performance:** Enhances visual clarity of boundaries but increases rendering time due to additional gradient calculations and shading modifications [S3].
- **Strengths:** Improves readability of complex anatomical structures, especially in medical visualization [S1][S2, unofficial][S3]. Effective for highlighting tissue boundaries and silhouettes [S2, unofficial].
- **Limitations:** May introduce artifacts if parameters like $v_{bump}$ are not tuned correctly [S3]. Requires appropriate transfer function design to avoid over-emphasizing noise [G].
- **When to use:** Medical illustration, anatomical visualization, and scenarios where boundary clarity is critical [S2, unofficial][S3].
- **When not to use:** When real-time performance is paramount and boundary detail is less important than overall structure [S3].

## Background
Boundary enhancement originated as a technique to improve the visibility of object boundaries in volume rendering [S1]. It has been widely adopted in medical illustration and visualization, often in conjunction with silhouette techniques [S2, unofficial]. The method has evolved to include variants like temporal gradient enhancement for 4D data [S1] and bump mapping for enhanced surface perception [S3]. It is commonly used in surgical simulation, anatomical illustration, and neuro-visualization [S2, unofficial][S3].

## Sources
[S1] Towards Advanced Volumetric Display of the Human Musculoskeletal System, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/021-025, fetched 2026-10-26
[S2] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
[S3] High-Quality Multimodal Volume Visualization of Intracerebral Pathological Tissue, https://doi.org/10.2312/vcbm/vcbm08/167-176, fetched 2026-10-26
