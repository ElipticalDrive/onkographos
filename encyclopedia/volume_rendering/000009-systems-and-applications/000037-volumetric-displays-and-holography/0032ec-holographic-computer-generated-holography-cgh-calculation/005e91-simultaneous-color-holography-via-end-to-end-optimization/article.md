# Simultaneous Color Holography via End-to-End Optimization

Also called: Simultaneous RGB Holography, Single SLM Pattern RGB Holography

## Overview
This method generates a full RGB hologram using a single Spatial Light Modulator (SLM) pattern illuminated simultaneously by red, green, and blue light sources, eliminating the need for field sequential color or multiple SLMs [S1]. It sets itself apart from Depth Division Multiplexing by directly optimizing the output hologram end-to-end rather than assigning colors to different depths, and from Bit Division Multiplexing by using a flexible framework with a perceptual loss function for direct optimization rather than a two-step lookup table process [S1].

## How it works
The method formulates hologram generation as an end-to-end differentiable optimization problem. It maps target RGB intensities to SLM voltage values by minimizing a perceptual loss function through gradient descent [S1].

**Inputs and Outputs**
*   **Input:** Target RGB intensity image $I_{target}(x,y)$; SLM physical parameters (wavelengths $\lambda_R, \lambda_G, \lambda_B$; phase range).
*   **Output:** SLM voltage matrix $V(x,y)$; Reconstructed RGB complex wavefield at the sensor plane.

**Data Structures**
*   **SLM Voltage Matrix:** 2D array of floating-point values representing digital input levels to the SLM.
*   **Complex Wavefield:** 2D array of complex numbers representing amplitude and phase at the sensor plane for each color channel.
*   **Perceptual Loss Weights:** Scalars or matrices weighting luminance and chrominance channels based on human visual acuity [S1].

**Steps**
1.  **SLM Modeling:** Map the current SLM voltage values $V$ to a complex field $U_{SLM}$ at the SLM plane. This model accounts for the SLM's wavelength-dependent response and cross-talk. It uses a learned cross-talk kernel and a linear lookup table to map voltage to complex amplitude [S1].
    $$ U_{SLM} = \text{SLMModel}(V, \lambda) $$
2.  **Propagation:** Propagate the complex wavefield from the SLM plane to the sensor plane. This uses a modified propagation model [Gopakumar et al. 2021] that separates the zeroth and first diffraction orders and combines them through a U-Net architecture to simulate the optical system's point spread function [S1].
    $$ U_{sensor} = \text{PropagationModel}(U_{SLM}, \lambda) $$
3.  **Loss Calculation:** Compute the color-specific perceptual loss between the reconstructed intensity $I_{recon}$ and the target intensity $I_{target}$. The loss function incorporates human color perception, specifically the difference in visual acuity between chrominance and luminance channels [S1].
    $$ L = \text{PerceptualLoss}(I_{recon}, I_{target}) $$
4.  **Gradient Update:** Calculate gradients of the loss with respect to the SLM voltages using automatic differentiation (e.g., PyTorch’s autograd) [S1].
    $$ \frac{\partial L}{\partial V} = \text{Autograd}(L, V) $$
5.  **Optimization:** Update the SLM voltages $V$ using a gradient-based optimizer (e.g., Adam or SGD) to minimize the loss [S1].
    $$ V_{new} = V_{old} - \eta \frac{\partial L}{\partial V} $$
    Repeat steps 1-5 until convergence or a maximum number of iterations is reached.

**Parameters**
*   **SLM Phase Range:** The method leverages the extended phase range of Liquid Crystal on Silicon (LCoS) SLMs, up to $10\pi$, to create diversity in phase triplets for RGB [S1].
*   **Perceptual Loss Weights:** Weights for luminance and chrominance channels based on human visual acuity [S1].
*   **Learning Rate ($\eta$):** Step size for gradient descent [G].
*   **Optimization Iterations:** Number of optimization steps [G].

**Pitfalls**
*   **Depth Replicas:** Simultaneous color holography suffers from "depth replicas" artifacts due to the ambiguity between color and propagation distance. This is mitigated by using the extended phase range of the SLM [S1].
*   **Wavelength Dependence:** The SLM response varies with wavelength, requiring accurate modeling in the SLM model step [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_hologram(target_rgb, slm_params, wavelengths, num_iterations):
    """
    End-to-end optimization for simultaneous color holography.
    
    Args:
        target_rgb: Target RGB intensity image (H, W, 3)
        slm_params: Dictionary containing SLM physical parameters
        wavelengths: Tuple of wavelengths (lambda_R, lambda_G, lambda_B)
        num_iterations: Number of optimization steps
        
    Returns:
        optimized_voltages: Optimized SLM voltage matrix (H, W)
    """
    # Initialize SLM voltages randomly or with a baseline pattern [G]
    voltages = torch.randn(H, W, requires_grad=True)
    
    for i in range(num_iterations):
        voltages.retain_grad()
        
        # Step 1: SLM Model
        # Maps voltage to complex field using learned cross-talk kernel and LUT
        # Assumes slm_model is a differentiable module defined elsewhere
        complex_field_slm = slm_model(voltages, wavelengths)
        
        # Step 2: Propagation Model
        # Propagates wavefield to sensor plane using modified Gopakumar et al. model
        # Assumes propagation_model is a differentiable module defined elsewhere
        complex_field_sensor = propagation_model(complex_field_slm, wavelengths)
        
        # Compute reconstructed intensity from complex field
        reconstructed_rgb = torch.abs(complex_field_sensor) ** 2
        
        # Step 3: Perceptual Loss
        # Loss function based on human color perception (luminance/chrominance acuity)
        # Assumes perceptual_loss is a differentiable function defined elsewhere
        loss = perceptual_loss(reconstructed_rgb, target_rgb)
        
        # Step 4: Gradient Calculation
        loss.backward()
        
        # Step 5: Update Voltages
        # Using a simple gradient descent step; in practice, use Adam/SGD
        with torch.no_grad():
            voltages -= learning_rate * voltages.grad
            
        # Clear gradients for next iteration
        voltages.grad.zero_()
        
    return voltages.detach()
```

## Operating characteristics
*   **Time Complexity:** The optimization process is iterative and computationally expensive, scaling with the number of optimization iterations and the complexity of the U-Net propagation model [S1].
*   **Memory Complexity:** Requires storing the SLM voltage matrix, complex wavefields at each step, and gradients for backpropagation [G].
*   **Strengths:** Eliminates the need for field sequential color or multiple SLMs; allows for extended phase range utilization; improves color fidelity via perceptual loss [S1].
*   **Limitations:** Computationally intensive optimization; sensitive to accurate SLM and propagation models; potential for depth replica artifacts if phase range is not sufficiently extended [S1].
*   **When to use:** When a single SLM pattern is required for simultaneous RGB illumination, and computational resources for optimization are available [S1].
*   **When not to use:** When real-time hologram generation is required without pre-computation; when SLM phase range is limited to $2\pi$ [S1].

## Background
This method was introduced by researchers at Reality Labs Research, Meta, USA [S1]. It addresses the limitations of traditional color holography methods such as field sequential color, depth division multiplexing, and bit division multiplexing [S1]. The work builds on prior research in differentiable holography and perceptual loss functions for image optimization [S1]. It was validated experimentally on 2D and 3D content using a custom camera-calibrated model [S1].

## Sources
[S1] Simultaneous Color Computer Generated Holography, https://doi.org/10.48550/arxiv.2303.11287, fetched 2026-10-30
