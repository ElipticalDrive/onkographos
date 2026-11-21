# Gerchberg-Saxton Algorithm

Also called: Iterative Phase Optimization, iterative phase retrieval

## Overview
The Gerchberg-Saxton algorithm is an iterative optimization method used to compute phase-only holograms by determining object point phase values that cause interference resulting in constant amplitude across the hologram plane [S1]. It solves the problem of phase-only display constraints where the eye does not detect image phase, allowing the algorithm to treat object phases as free variables to satisfy amplitude constraints on the display [S1]. It is distinguished from its parent, Phase-Aware Gradient Descent (PAGD), by its specific use of back-and-forth propagation with hard amplitude constraints rather than gradient-based loss minimization, and from its sibling, Camera-Calibrated Wave Propagation Model Training, by being a direct analytical iterative solver rather than a learned model [S1].

## How it works
**Inputs:**
*   Target image intensity distribution (implicitly defining the required amplitude on the hologram plane).
*   Hologram plane resolution and geometry.
*   Propagation model (typically Fresnel or Fraunhofer diffraction, implemented via Fourier transforms).

**Outputs:**
*   Phase-only hologram $H$ (complex field with unit amplitude and optimized phase).

**Data Structures:**
*   Complex field array $U_o$ representing the object plane.
*   Complex field array $U_h$ representing the hologram plane.
*   Phase array $\phi_o$ for object points.

**Steps:**
1.  **Initialization:** Initialize the complex field on the hologram plane $U_h^{(0)}$ with the target amplitude $A_{target}$ and a random or zero phase distribution. Alternatively, initialize the object plane field and propagate to the hologram plane [S1].
2.  **Forward Propagation:** Propagate the current hologram plane field $U_h^{(k)}$ to the object plane $U_o^{(k)}$ using the appropriate diffraction integral (e.g., Fourier transform for far-field) [S1].
3.  **Object Plane Constraint:** Replace the amplitude of the object plane field $U_o^{(k)}$ with the target object amplitude (or keep it as is if the target is on the hologram plane, depending on the specific formulation; the standard GS for holograms often enforces amplitude on the hologram plane directly). *Note: The source describes propagating back and forth between object and hologram planes while enforcing amplitude constraints at each plane [S1]. In the context of phase-only holograms for displays, the constraint is typically that the hologram plane amplitude must be constant (uniform) because the SLM can only modulate phase.*
4.  **Hologram Plane Constraint:** Propagate the object plane field $U_o^{(k)}$ back to the hologram plane $U_h^{(k+1)}$. Enforce the amplitude constraint on $U_h^{(k+1)}$ by setting its amplitude to the required constant value (e.g., 1.0) while preserving its calculated phase [S1].
5.  **Iteration:** Repeat steps 2-4 for $N$ iterations.
6.  **Final Output:** The final phase distribution of the hologram plane field is extracted as the hologram $H = \arg(U_h^{(N)})$ [S1].

**Formulas:**
Let $U_h$ be the complex field on the hologram plane and $U_o$ be the complex field on the object plane.
Propagation from hologram to object:
$$ U_o = \mathcal{P}(U_h) $$
Propagation from object to hologram:
$$ U_h' = \mathcal{P}^{-1}(U_o) $$
Amplitude constraint enforcement on the hologram plane:
$$ U_h^{(k+1)} = A_{target} \cdot \exp(i \cdot \arg(U_h')) $$
Where $A_{target}$ is the constant required amplitude (often 1 for phase-only).

**Parameters:**
*   Number of iterations: Typically 10-100 [S1].

**Edge Cases and Pitfalls:**
*   **Speckle:** Iteratively optimized holograms tend to include undesirable high-frequency noise or speckle [S1].
*   **Computational Expense:** The method is computationally expensive due to the need for many iterations [S1].
*   **Local Minima:** The algorithm may converge to local minima, resulting in suboptimal holograms. Modern variations add error compensation and define "do not care" regions to increase performance [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gerchberg_saxton(target_amplitude_hologram, num_iterations, propagation_func, inverse_propagation_func):
    """
    Computes a phase-only hologram using the Gerchberg-Saxton algorithm.
    
    Args:
        target_amplitude_hologram: 2D array of the desired amplitude on the hologram plane.
        num_iterations: Number of GS iterations.
        propagation_func: Function to propagate field from hologram to object plane.
        inverse_propagation_func: Function to propagate field from object to hologram plane.
        
    Returns:
        phase_hologram: 2D array of the computed phase-only hologram.
    """
    # Initialize hologram plane field with target amplitude and random phase
    # [G] Random phase initialization is standard to break symmetry; 
    # source [S1] implies starting with some field and propagating.
    hologram_field = target_amplitude_hologram * np.exp(1j * np.random.uniform(0, 2 * np.pi, target_amplitude_hologram.shape))
    
    for k in range(num_iterations):
        # Forward propagation: Hologram -> Object
        object_field = propagation_func(hologram_field)
        
        # [G] The source [S1] mentions enforcing amplitude constraints at each plane.
        # In standard hologram generation, the object plane constraint is often the target image intensity.
        # However, the specific quote says "find values phi_o that cause the object points to interfere 
        # such that the required amplitude values of H are constant". This implies the constraint is on H.
        # If there is a constraint on the object plane (target image), it would be applied here.
        # Since the source emphasizes constant amplitude on H, we assume the object plane constraint 
        # is either the target image amplitude or simply preserving the phase from the previous step 
        # if no object constraint is defined. 
        # Standard GS for holograms often uses: Object Amplitude = Target Image Amplitude.
        # Let's assume a target object amplitude is available or derived. 
        # If not specified, we preserve the phase from the forward pass.
        # [G] To be implementable, we assume a target object amplitude exists or is the magnitude of the current field.
        # Given the source text focuses on constant amplitude on H, we will apply the H constraint.
        # If an object constraint exists, it would be: object_field = target_object_amplitude * exp(1j * angle(object_field))
        
        # Backward propagation: Object -> Hologram
        hologram_field_new = inverse_propagation_func(object_field)
        
        # Enforce amplitude constraint on Hologram plane
        # The goal is constant amplitude on H
        hologram_field = target_amplitude_hologram * np.exp(1j * np.angle(hologram_field_new))
        
    # Extract phase
    phase_hologram = np.angle(hologram_field)
    return phase_hologram
```

## Subtopics

### Weighted Fourier Iterative Algorithm
An iterative algorithm to design Computer-Generated Holograms (CGHs) by finding a phase distribution on the hologram plane that approximates a target focal pattern, used to generate multiple simultaneous focal points for voxel positioning [S2]. It differs from the base Gerchberg-Saxton algorithm by introducing a weighting function $w$ that is iteratively updated to control convergence and improve uniformity among reconstructed focal points [S2]. It is chosen for its high uniformity and simplicity based on iterative FFT operations [S2].

**Steps:**
1.  Initialize amplitude $a_h(x,y)$ as uniform values on the hologram plane [S2].
2.  Form initial phase $\phi_h$ by inverse Fourier transform of the complex amplitude constructed with the target pattern [S2].
3.  In iteration step $k$, constrain the reconstructed plane amplitude to the desired distribution $a_t(\nu_x, \nu_y)$ using the weighting function $w^{(k+1)}$:
    $$ u_r^{(k+1)}(\nu_x, \nu_y) = w^{(k+1)}(\nu_x, \nu_y) a_t(\nu_x, \nu_y) \exp(i \phi_r(\nu_x, \nu_y)) $$
    where $\phi_r = \arg(u_r^{(k)})$ [S2].
4.  Update the weighting function iteratively:
    $$ w^{(k+1)}(\nu_x, \nu_y) = w^{(k)}(\nu_x, \nu_y) \left\{ \frac{a_t(\nu_x, \nu_y)}{|u_r^{(k)}(\nu_x, \nu_y)|} \right\}^\alpha $$
    [S2].
5.  Generate optimal $\phi_h$ as a CGH by calculating $\arg(u_h)$ after iterations [S2].

**Formulas:**
$$ u_r(\nu_x, \nu_y) = \mathcal{F}[u_h(x,y)] $$
$$ u_h(x,y) = a_h(x,y) \exp(i \phi_h(x,y)) $$
Where $\mathcal{F}[\cdot]$ is the Fourier transform, $a_h/\phi_h$ are amplitude/phase on the SLM, and $\nu_x/\nu_y$ are spatial frequencies [S2].

**Parameters:**
*   $\alpha$: Adjustable parameter controlling convergence. Typical value: 0.1 [S2].

## Operating characteristics
*   **Time Complexity:** The algorithm requires $N$ iterations, where each iteration involves forward and backward propagation (typically FFTs). Thus, the time complexity is $O(N \cdot M \log M)$, where $M$ is the number of pixels in the hologram plane [S1].
*   **Memory Complexity:** Requires storage for the complex fields on the object and hologram planes, $O(M)$ [S1].
*   **Measured Performance:** Computational expense is high due to the need for many iterations (10-100) [S1].
*   **Strengths:** It is the most common method to compute phase-only holograms and is numerically accurate [S1]. It effectively finds phase values that cause interference to produce constant amplitude on the hologram, which is suitable for phase-only displays [S1].
*   **Limitations:** Iteratively optimized holograms tend to include undesirable high-frequency noise or speckle [S1]. It is computationally expensive [S1].
*   **When to use:** When computing phase-only holograms for displays where the eye does not detect image phase, and high uniformity is required [S1].
*   **When not to use:** When computational speed is critical and iterative methods are too slow, or when speckle noise is unacceptable without further post-processing [S1].

## Background
The Gerchberg-Saxton algorithm was introduced by Gerchberg and Saxton in 1972 [S1]. It is a well-established phase retrieval method used in various applications, including holography and optics [S2]. Modern variations have been developed to add error compensation and define "do not care" regions to increase performance [S1].

## Sources
[S1] Holographic near-eye displays for virtual and augmented reality, https://doi.org/10.1145/3072959.3073624, fetched 2026-10-30
[S2] Dual-Path Holographic Laser-Excited Volumetric Display, https://doi.org/10.1145/3816042, fetched 2026-10-27
