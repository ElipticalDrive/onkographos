# Simultaneous Color Focal Stack Optimization

Also called: Learned Wave Propagation SLM Optimization, Multi-plane Focal Stack Optimization

## Overview
Simultaneous Color Focal Stack Optimization is a method for generating computer-generated holograms (CGH) that reconstruct a full-color image across multiple depth planes using a single spatial light modulator (SLM) pattern illuminated by simultaneous red, green, and blue light sources [S1]. It solves the problem of creating volumetric color displays without the complexity of field-sequential color or multiple SLMs [S1]. Unlike methods that optimize for a single focal plane, this approach uses a learned wave propagation model and gradient descent to jointly optimize the SLM pattern for a stack of target intensities at different depths [S1]. It builds on [Simultaneous Color Holography via End-to-End Optimization](17688-simultaneous-color-holography-via-end-to-end-optimization.md) by extending the optimization target from a single plane to a multi-plane focal stack [S1].

## How it works
**Inputs:**
*   Learned wave propagation model $Holo(\cdot)$ for the unfiltered holography system [S1].
*   Initial SLM pattern $P_{SLM}$ [S1].
*   Target intensity images $I_{target}[d]$ for each depth plane $d$ in the focal stack [S1].

**Outputs:**
*   Optimized SLM pattern $P_{SLM}$ [S1].
*   Captured simultaneous color focal stack $FocalStack$ [S1].

**Data Structures:**
*   $P_{SLM}$: The complex or phase-only pattern on the SLM. The specific bit depth and phase range constraints are discussed in [SLM Bit Depth Analysis](#slm-bit-depth-analysis).
*   $I_{target}[d]$: A 2D image array representing the desired intensity at depth $d$.
*   $FocalStack$: A list or array of captured images, one for each depth plane.

**Algorithm:**

1.  **Initialization:** Load the learned wave propagation model. Initialize $P_{SLM}$ and define $I_{target}$ for each depth plane in the focal stack [S1].
2.  **Optimization Loop:** Iterate while $P_{SLM}$ has not converged [S1].
    3.  **Plane-wise Optimization:** For each depth plane $d$ in the focal stack hologram:
        4.  **Gradient Clearing:** Clear previous gradients with respect to $P_{SLM}$ [S1].
        5.  **Forward Propagation:** Generate the optical field from the current $P_{SLM}$ and propagate it to depth plane $d$ using the learned model:
            $$ Field_{out} = Holo(P_{SLM}, \text{depth}=d) $$
            [S1]
        6.  **Loss Computation:** Compute the loss using a custom perceptual loss function:
            $$ Loss = LossFunc(Field_{out}, I_{target}[d]) $$
            [S1]
        7.  **Backpropagation:** Backpropagate the gradient of $Loss$ with respect to $P_{SLM}$ [S1].
        8.  **Update:** Update $P_{SLM}$ using the calculated gradients (e.g., via gradient descent) [S1].
9.  **Capture:** Experimentally capture the simultaneous color focal stack [S1].
    10. For each depth plane $d$ in the focal stack hologram:
        11. Move the camera to depth plane $d$ [S1].
        12. Capture the hologram image:
            $$ Hologram = CaptureImage(camera) $$
            [S1]
        13. Add the captured hologram to the focal stack:
            $$ FocalStack.append(Hologram) $$
            [S1]
14. **Output:** Return the optimized $P_{SLM}$ and the captured $FocalStack$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SimultaneousColorFocalStackOptimization(learned_model, initial_p_slm, target_intensities):
    """
    Optimizes an SLM pattern for a multi-plane focal stack using a learned wave propagation model.
    
    Args:
        learned_model: A callable Holo(P_SLM, depth) that propagates the field to a specific depth.
        initial_p_slm: The initial SLM pattern (tensor/array).
        target_intensities: A list or dictionary of target intensity images I_target[d] for each depth plane d.
        
    Returns:
        optimized_p_slm: The optimized SLM pattern.
        focal_stack: A list of captured images for each depth plane.
    """
    # 1. Initialize
    p_slm = initial_p_slm
    focal_stack = []
    
    # 2. Optimize
    while not converged(p_slm):
        for d in target_intensities.keys():
            # 3. Clear gradients
            clear_gradients(p_slm)
            
            # 4. Forward propagation
            # Field_out is the complex field at depth d
            field_out = learned_model(p_slm, depth=d)
            
            # 5. Compute loss
            # LossFunc is the custom perceptual loss function defined in [Perceptual Loss Function for Color Holography]
            loss = LossFunc(field_out, target_intensities[d])
            
            # 6. Backpropagation
            loss.backward()
            
            # 7. Update SLM pattern
            # Update rule depends on the optimizer (e.g., SGD, Adam)
            update_parameters(p_slm, learning_rate=0.01) # [G] Typical learning rate placeholder; specific optimizer not given in sources.
            
    # 8. Capture (Experimental Step)
    # Note: This step requires physical hardware interaction.
    # In a pure simulation context, this would be replaced by evaluating the field at each depth.
    for d in target_intensities.keys():
        # Move camera to depth plane d
        move_camera_to_depth(d)
        
        # Capture hologram
        hologram = capture_image()
        
        # Add to focal stack
        focal_stack.append(hologram)
        
    return p_slm, focal_stack

def converged(p_slm):
    """
    Check for convergence. Not explicitly defined in sources. [G]
    Typically checks for change in loss or pattern magnitude below a threshold.
    """
    # Placeholder for convergence check logic
    return False 
```

**Parameters:**
*   **Convergence Criteria:** Not given in the sources. [S1]
*   **Learning Rate:** Not given in the sources. [S1]
*   **Optimizer:** Not given in the sources. [S1]

**Edge Cases and Pitfalls:**
*   **Convergence Failure:** If the optimization fails to converge, or if the product of initial source power and weighting function exceeds 100% (in the context of Source Power Optimization), the exposure time should be increased and the source power optimization restarted [S1].
*   **Bit Depth Quantization:** Significant drop-off in image quality (PSNR and SSIM) occurs between 5 and 6 bits. An effective bit depth of at least 6 bits is required for adequate image quality [S1].

## Subtopics

### Source Power Optimization
An active camera-in-the-loop iterative process to optimize the power of each color channel of the SLED source for color fidelity [S1]. It is chosen when precise color balance is required and the source power cannot be perfectly calibrated beforehand [S1]. It sets the source power to an arbitrary value (<100%), captures a baseline reference image, debayers it, and maps it to SLM space [S1]. Three learnable weighting parameters (one per color channel) are initialized to unity and applied to the reference image as a proxy for source power [S1]. The process iteratively captures an image, calculates loss against the target, and backpropagates through the weighting parameters to update the initial source power [S1]. It repeats until the color weighting parameters converge (typically 15-30 iterations) [S1]. If convergence fails or the product of initial source power and weighting function exceeds 100%, the exposure time is increased and the optimization restarted [S1].

### SLM Bit Depth Analysis
An analysis defining the effective bit depth as the number of bits contained in a $2\pi$ interval of the extended phase range [S1]. It determines the minimum quantization level required for adequate image quality [S1]. It simulates holograms by optimizing with gradient descent and then quantizing to the target bit depth [S1]. Significant drop-off in PSNR and SSIM is observed between 5 and 6 bits [S1]. The minimum effective bit depth required is 6 bits [S1]. For 6-bit SLM simulation, PSNR is 31.00 and SSIM is 0.856; for 8-bit, PSNR is 34.98 and SSIM is 0.943 [S1]. No image quality improvement is achieved for simultaneous color holograms once each color channel has a bit depth of at least 6 bits across a $2\pi$ phase range [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1]
*   **Memory Complexity:** Not given in the sources. [S1]
*   **Measured Performance:** For 6-bit SLM simulation, PSNR is 31.00 and SSIM is 0.856. For 8-bit SLM simulation, PSNR is 34.98 and SSIM is 0.943 [S1].
*   **Scaling:** Not given in the sources. [S1]
*   **Strengths:** Allows for the creation of volumetric color displays using a single SLM and simultaneous RGB illumination, eliminating the need for field-sequential color or multiple SLMs [S1]. Uses a learned model which may be faster or more accurate than analytical propagation for unfiltered systems [S1].
*   **Limitations:** Requires a learned wave propagation model which must be trained for the specific unfiltered holography system [S1]. Requires experimental capture for validation, which involves moving the camera [S1]. Sensitive to source power calibration, requiring active optimization [S1].
*   **When to use:** When creating volumetric color holograms with a single SLM and simultaneous RGB illumination is desired, and a learned propagation model is available [S1].
*   **When not to use:** When high bit-depth SLMs are not available (minimum 6 bits effective required) [S1]. When real-time performance is critical and the optimization time is prohibitive (not given) [S1].

## Background
The method was introduced by Markley et al. in "Simultaneous Color Computer Generated Holography" [S1]. It addresses the challenges of near-eye holographic displays and volumetric imaging by enabling simultaneous color reconstruction across multiple depth planes [S1]. The approach relies on a learned wave propagation model for the unfiltered holography system, which accounts for system-specific aberrations and filtering effects [S1]. The research highlights the importance of source power optimization and SLM bit depth analysis for achieving high-fidelity color holograms [S1].

## Sources
[S1] Simultaneous Color Computer Generated Holography, https://doi.org/10.48550/arxiv.2303.11287, fetched 2026-10-30
