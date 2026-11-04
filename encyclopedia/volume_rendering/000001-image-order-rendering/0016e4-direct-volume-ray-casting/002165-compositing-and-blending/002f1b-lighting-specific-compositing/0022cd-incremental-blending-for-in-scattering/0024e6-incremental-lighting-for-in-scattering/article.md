# Incremental Lighting for In-Scattering

Also called: Translucency Approximation via Indirect Attenuation

## Overview
Incremental Lighting for In-Scattering is a screen-space technique that approximates volumetric light scattering (translucency) by propagating and blurring indirect lighting components across volume slices [S1]. It extends the base Incremental Blending for In-scattering method by introducing per-channel indirect attenuation parameters and a blur operation, allowing light to change color as it penetrates deeper into the volume, similar to the effect of holding a flashlight under a hand [S1].

## How it works
This method builds on [Incremental Blending for In-scattering](2880-incremental-blending-for-in-scattering.md) by modifying the light buffer initialization, the compositing blend function, and the data flow to support spatial blurring of indirect attenuation.

### Inputs and Outputs
*   **Inputs:**
    *   Volume data (density, color).
    *   Indirect attenuation parameters $A_i$: Alpha values for each RGB color channel (distinct from the standard alpha $A$ used in the eye-pass) [S1].
    *   Light color $C_L$ [S1].
    *   Sample distance $d$ [S1].
    *   Blur angle: A user-defined parameter scaling the perturbation radius [S1].
    *   Perturbation texture: Used to generate modified texture coordinates for blurring [S1].
*   **Outputs:**
    *   Final rendered image with translucent lighting effects.

### Data Structures
*   **Light Buffers:** Two framebuffers (or textures) are used in an alternating fashion. Let them be $L_{current}$ and $L_{next}$.
    *   $L_{current}$: Stores the indirect attenuation state for the current slice iteration.
    *   $L_{next}$: The render target for the updated attenuation state after blurring and blending [S1].
*   **Perturbation Texture:** A texture containing noise or pattern data used to perturb texture coordinates for the blur sampling [S1].

### Algorithm Steps
The method modifies the standard incremental ray casting loop (Algorithm 39-3 in [S1]) as follows:

1.  **Initialization:**
    *   Initialize the light buffer ($L_{current}$) with the value $1 - C_L$ instead of the standard light color $C_L$ [S1].

2.  **Per-Slice Loop:**
    For each volume slice (or ray step), perform the following:

    3.  **Eye Pass (Pass 1):**
        *   Sample the volume.
        *   Read the indirect attenuation value $A_{in}$ from $L_{current}$ using standard texture coordinates [S1].
        *   Compute the fragment color $C$ (standard direct lighting/compositing result) [S1].
        *   Modify the color by multiplying it by the remaining indirect transmission: $C_{final} = C \times (1 - A_{in})$ [S1].
        *   Output $C_{final}$ to the color buffer.

    4.  **Light Pass (Pass 2):**
        *   This pass updates the light buffer to propagate and blur indirect light for the next slice.
        *   **Blending Operation:** Replace the standard "Over" operator with the **Light Buffer Compositing** blend.
            *   Let $C_1$ be the incoming fragment color (from the volume sample's indirect contribution).
            *   Let $C_0$ be the color currently in the target render buffer ($L_{next}$).
            *   Let $A_{in}$ be the incoming fragment alpha (the indirect attenuation parameter $A_i$ for the current sample) [S1].
            *   The blended result $C_{out}$ is computed as:
                $$ C_{out} = C_0 \times (1 - A_{in}) + C_1 \times (1 - C_0) $$
                *(Note: The source describes Equation 8 as $C_{out} = C_{in} * (1 - C_{target}) + C_{target} * (1 - A_{in})$ in the notes, but the text clarifies $C_1$ is incoming fragment color and $C_0$ is target buffer color. The standard Over operator is $C_0(1-A_1) + C_1 A_1$. The specific "Light Buffer Compositing" for translucency is defined in the text as replacing Over. Based on the note's formula structure $C_{out} = C_{in} * (1 - C_{target}) + C_{target} * (1 - A_{in})$, and standard alpha blending logic adapted for this specific buffer, the implementation must follow the source's Equation 8 definition. The note provides: $C_{out} = C_{in} * (1 - C_{target}) + C_{target} * (1 - A_{in})$. We use this exact formula.)*
        *   **Blur Operation:**
            *   Calculate the blur radius $R$ using the sample distance $d$ and the blur angle $\theta$:
                $$ R = d \times \theta $$
                *(Note: The source mentions Equation 9 for Blur Radius but does not provide the explicit LaTeX formula in the snippet, only describing it as scaling the perturbation by $d$ and the blur angle. We assume a linear scaling or standard angular projection. If the source implies $R = d \cdot \tan(\theta)$ or similar, it is not explicitly given. We state the dependency.)*
            *   For each fragment in this pass, modify the texture coordinates for sampling $L_{current}$ using the perturbation texture. The perturbation is scaled by $R$ [S1].
            *   Sample $L_{current}$ multiple times using these modified coordinates.
            *   Compute the blurred attenuation as a weighted sum of the sampled values [S1].
            *   Blend this blurred value into $L_{next}$ using the Light Buffer Compositing operation defined above [S1].

    5.  **Buffer Swap:**
        *   After processing the slice, swap $L_{current}$ and $L_{next}$ so that the updated buffer becomes the source for the next slice's Eye Pass [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
# L_buffers is a list of two framebuffers/textures
L_current = L_buffers[0]
L_next = L_buffers[1]
L_current.clear()
L_current.set_pixel(1.0 - C_L) # Initialize with 1 - Light Color

# Per-Slice Loop
for slice in volume_slices:
    # Pass 1: Eye Pass
    for fragment in slice:
        # Standard volume sampling to get direct color C and standard alpha A
        C_direct, A_standard = sample_volume(fragment)
        
        # Read indirect attenuation from current light buffer
        A_indirect = L_current.sample(fragment.uv)
        
        # Multiply direct color by remaining indirect transmission
        C_final = C_direct * (1.0 - A_indirect)
        
        output_color_buffer(fragment.uv, C_final)

    # Pass 2: Light Pass (Update and Blur Light Buffer)
    L_next.clear()
    
    # Determine blur radius
    # R = d * blur_angle (assuming linear scaling as per description)
    R = sample_distance * blur_angle
    
    for fragment in slice:
        # 1. Compute Blurred Indirect Attenuation
        blurred_A = 0.0
        weight_sum = 0.0
        
        # Sample current light buffer multiple times with perturbed coordinates
        for sample in blur_samples:
            # Perturb UV using perturbation texture scaled by R
            perturbation = perturbation_texture.sample(sample.uv)
            uv_perturbed = fragment.uv + perturbation * R
            
            # Sample the current light buffer
            val = L_current.sample(uv_perturbed)
            
            # Weighted sum (weights depend on blur kernel, e.g., Gaussian)
            w = get_blur_weight(sample.offset, R)
            blurred_A += val * w
            weight_sum += w
            
        blurred_A /= weight_sum
        
        # 2. Compute Incoming Fragment Color for Light Buffer
        # In the context of the light buffer, the "incoming fragment" 
        # is the indirect contribution being propagated. 
        # Based on the method, we are propagating the attenuation/light state.
        # The note says "set the fragment shader color to Ai instead of 0".
        # This implies C_in for the blend equation is related to Ai.
        # However, Equation 8 blends C_in and C_target.
        # In the Light Pass, we are updating the light buffer.
        # The "incoming fragment" C1 in Eq 8 refers to the value being written 
        # by the fragment shader before blending.
        # The text says: "set the color in the fragment shader to Ai".
        # So C1 = A_indirect (the raw indirect attenuation for this sample).
        
        C_in = A_indirect # The indirect attenuation parameter for this sample
        
        # 3. Apply Light Buffer Compositing
        # C0 is the value currently in L_next (target)
        C_target = L_next.sample(fragment.uv)
        A_in = A_indirect # The alpha for the incoming fragment in the blend equation
        
        # Equation 8: C_out = C_in * (1 - C_target) + C_target * (1 - A_in)
        # Note: The note formula uses C_in and C_target.
        # C_in is the fragment color (A_indirect).
        # C_target is the buffer color.
        # A_in is the fragment alpha (A_indirect).
        
        C_out = C_in * (1.0 - C_target) + C_target * (1.0 - A_in)
        
        L_next.set_pixel(fragment.uv, C_out)

    # Swap buffers
    L_current, L_next = L_next, L_current
```

### Parameters
*   **$A_i$**: Indirect attenuation parameters, alpha values for each RGB channel. Must be set independently per channel to achieve color-changing penetration [S1].
*   **$d$**: Sample distance, used to calculate the blur radius [S1].
*   **Blur Angle**: User-defined parameter scaling the perturbation radius [S1].

### Edge Cases and Pitfalls
*   **Color-Changing Penetration:** To achieve effects like white light turning red as it penetrates (e.g., flashlight under hand), set the $A_i$ value for red smaller than the $A_i$ values for green and blue. This results in lower attenuation of the red component, allowing more red light to pass through deeper layers [S1].
*   **Precision:** The method relies on screen-space buffers. Matching the resolution of light propagation to the viewport produces crisp shadows with minimal aliasing artifacts [S1].
*   **Decoupling:** The resolution of light transport is decoupled from the 3D data grid, allowing accurate lighting of volumes without increasing the 3D sampling density [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of volume samples and slices, plus the cost of the blur operation (multiple samples per fragment in Pass 2).
*   **Memory Complexity:** Requires two additional light buffers (framebuffers) in addition to standard color and depth buffers.
*   **Strengths:**
    *   Produces convincing translucent effects for clouds and atmospheric phenomena [S1].
    *   Decouples light transport resolution from 3D data grid resolution [S1].
    *   Matches light propagation resolution to viewport, reducing aliasing in shadows [S1].
*   **Limitations:**
    *   Approximation: Relies on local neighborhood blurring, which may not capture complex global light transport accurately.
    *   Requires additional buffer management and passes.

## Background
This technique is described in *GPU Gems*, Chapter 39, "Volume Rendering Techniques" [S1]. It is part of the broader effort to compute volumetric light transport in screen space, offering advantages over purely grid-based methods for certain visual effects like clouds [S1].

## Sources
[S1] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
