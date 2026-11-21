# Forward-Scattered Intensity Transport

Also called: Light Buffer Intensity Calculation

## Overview
Forward-Scattered Intensity Transport is a method for approximating in-scattering (translucency) in volume rendering by propagating light intensity through a volume using a dedicated light buffer. It solves the problem of computing indirect illumination in a single-pass rendering pipeline by discarding backward-directed light relative to the light source, thereby accommodating the lack of knowledge about future slices during a front-to-back traversal. It differs from full scattering models by ignoring back-scattering during the intensity transport phase, which simplifies the hardware implementation at the cost of accuracy in dense, deep regions of the volume.

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data (density $\alpha$, scattering properties), light direction $\vec{m}_{\Omega_{lt}}$, sampling step size $\Delta$, and the number of expected scattering events $n(\vec{s})$ at the current sample position $\vec{s}$.
*   **Output:** Updated light intensity $I(\vec{s} + \vec{m}_{\Omega_{lt}}\Delta)$ and scattering event count for the next slice, stored in the light buffer.

**Data Structures:**
1.  **Light Buffer:** A texture storing two components per sample:
    *   **RGB Channel:** Light intensity $I(\vec{s})$.
    *   **Alpha Channel:** The number of scattering events $n(\vec{s})$ that have occurred up to this point.
2.  **Phase Function Texture:** Floating-point textures storing high-dynamic range multiple scattering phase functions $A^{ms}_\lambda^i(\Omega)$ for all particle types.
    *   **Resolution:** Approximately 0.18 degrees.
    *   **Depth:** Up to 26 convolutions (scattering events).
    *   **Content:** Stores $A^{ms}_\lambda^i(\Omega)$ and the pre-computed forward-scattered factors $F_i^\lambda$. The light distribution is considered essentially isotropic after 26 convolutions.

**Parameters:**
*   $\beta_{\lambda ex}$: Expected number of scattering events per unit distance for wavelength $\lambda$.
*   $\Delta$: Step size between sampling slices.
*   $T(\vec{s}_1, \vec{s}_2)$: Transmittance from $\vec{s}_1$ to $\vec{s}_2$.

**Method Steps:**

1.  **Calculate Forward-Scattered Contribution ($F_i^\lambda$):**
    For each scattering event $i$, calculate the integral of the phase function over the forward hemisphere (relative to the light direction). Backward-directed light is discarded to ensure single-pass compatibility.
    $$
    F_i^\lambda = \int_{\text{forward hemisphere}} A^{ms}_\lambda^i(\Omega) d\Omega \quad (23)
    $$
    These values are pre-computed and stored in the phase function texture.

2.  **Retrieve Current State:**
    Sample the light buffer at the previous slice position $\vec{s}$ to retrieve the current intensity $I(\vec{s})$ and the accumulated scattering count $n(\vec{s})$.

3.  **Compute Transmittance:**
    Calculate the transmittance $T(\vec{s}, \vec{s} + \vec{m}_{\Omega_{lt}}\Delta)$ from the current sample to the next sample along the light direction.

4.  **Update Intensity (Hardware-Efficient Approximation):**
    The exact discrete intensity update involves a product over the range of scattering events, which is impractical for graphics hardware. Instead, use the worst-case attenuation approximation using a single convolution index. This avoids linear interpolation of floating-point textures, which was unavailable on the target hardware.

    $$
    I(\vec{s} + \vec{m}_{\Omega_{lt}}\Delta) = I(\vec{s})T(\vec{s}, \vec{s} + \vec{m}_{\Omega_{lt}}\Delta) + I(\vec{s})(1 - T(\vec{s}, \vec{s} + \vec{m}_{\Omega_{lt}}\Delta)) \alpha(\vec{s}) F_{\lfloor n(\vec{s}) + 1 + \beta_{\lambda ex}\Delta \rfloor}^\lambda \quad (25)
    $$

    Where:
    *   $\alpha(\vec{s})$ is the density/scattering coefficient at $\vec{s}$.
    *   The index $\lfloor n(\vec{s}) + 1 + \beta_{\lambda ex}\Delta \rfloor$ determines which pre-computed $F$ factor to use.

5.  **Update Scattering Count:**
    Increment the scattering event count for the next slice. The new count $n(\vec{s} + \vec{m}_{\Omega_{lt}}\Delta)$ is updated based on the expected scattering events over the step $\Delta$.

6.  **Store Results:**
    Write the new intensity $I(\vec{s} + \vec{m}_{\Omega_{lt}}\Delta)$ to the RGB channel and the new scattering count to the Alpha channel of the light buffer at the next slice position.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_scattered_intensity_transport(
    light_buffer, 
    phase_function_texture, 
    current_pos, 
    light_direction, 
    step_size, 
    beta_lambda_ex,
    alpha_density
):
    """
    Updates light intensity and scattering count for the next slice.
    
    Args:
        light_buffer: Texture containing intensity (RGB) and scattering count (A).
        phase_function_texture: Texture containing pre-computed F factors.
        current_pos: Current sample position vector.
        light_direction: Normalized vector pointing towards the light source.
        step_size: Distance to the next sample.
        beta_lambda_ex: Expected scattering events per unit distance.
        alpha_density: Density/scattering coefficient at current_pos.
        
    Returns:
        next_intensity: Float, the updated light intensity.
        next_scattering_count: Int, the updated number of scattering events.
    """
    
    # 1. Retrieve current state from light buffer
    current_intensity = light_buffer.sample(current_pos).rgb
    current_scattering_count = light_buffer.sample(current_pos).a
    
    # 2. Calculate transmittance from current to next position
    next_pos = current_pos + light_direction * step_size
    transmittance = compute_transmittance(current_pos, next_pos)
    
    # 3. Determine the index for the F factor
    # Using the hardware-efficient approximation (Equation 25)
    # Index is floor(n(s) + 1 + beta * delta)
    f_index = floor(current_scattering_count + 1 + beta_lambda_ex * step_size)
    
    # 4. Retrieve the pre-computed forward-scattered factor F_i^lambda
    # F factors are stored in the phase function texture
    f_factor = phase_function_texture.sample_f_factor(f_index)
    
    # 5. Apply the intensity update formula (Equation 25)
    # I(s + m*delta) = I(s)*T + I(s)*(1-T)*alpha*F_index
    next_intensity = (
        current_intensity * transmittance + 
        current_intensity * (1.0 - transmittance) * alpha_density * f_factor
    )
    
    # 6. Update scattering count
    # The count increases by the expected number of scattering events over the step
    next_scattering_count = current_scattering_count + floor(beta_lambda_ex * step_size)
    
    return next_intensity, next_scattering_count
```

**Edge Cases and Pitfalls:**
*   **Linear Interpolation:** Linear interpolation is not available for floating-point textures on the target hardware. This necessitates the use of the worst-case attenuation approximation (Equation 25) instead of interpolating between scattering events.
*   **Backward Light:** Backward-directed light is discarded during intensity transport. This is acceptable because phase functions have weak back-scattered components and the system performs a single slicing pass. However, this approximation introduces errors that increase with the number of scattering events, making it least accurate in dense inner regions of the cloud.
*   **Final Gather:** While backward light is discarded during transport, light may still be backscattered towards the eye in the final gather step (Section 4.2.3 of the source), where the angular distribution function for one more scattering event is applied.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of sampling slices along the ray. The method performs a single pass through the volume.
*   **Memory Complexity:** Requires storage for the light buffer (intensity + scattering count) and the phase function texture (high-dynamic range, up to 26 convolutions).
*   **Performance:** The method is designed for hardware efficiency, avoiding complex products and floating-point interpolation. It achieves real-time performance (e.g., 3.0 fps for 512x512 resolution with 256 sampling planes) on the target hardware.
*   **Strengths:** Enables single-pass rendering of atmospheric phenomena (rainbows, halos, glory, aerial perspective) with reasonable accuracy in optically thin regions and for the first few scattering events.
*   **Limitations:** Accuracy degrades in dense, deep regions of the volume due to the approximation of backward light and the use of worst-case attenuation. The approximation error increases with the number of scattering events.

## Background
Forward-Scattered Intensity Transport was introduced in the paper "Efficient Rendering of Atmospheric Phenomena" [S1]. It is part of a system designed to capture essential optical effects in the atmosphere, such as rainbows, halos, the glory, and sunrise/sunset effects. The method was developed to address the limitations of previous techniques in handling multiple scattering and angular distribution efficiently on graphics hardware.

## Sources
[S1] Efficient Rendering of Atmospheric Phenomena, https://doi.org/10.2312/egwr/egsr04/375-386, fetched 2026-10-26
