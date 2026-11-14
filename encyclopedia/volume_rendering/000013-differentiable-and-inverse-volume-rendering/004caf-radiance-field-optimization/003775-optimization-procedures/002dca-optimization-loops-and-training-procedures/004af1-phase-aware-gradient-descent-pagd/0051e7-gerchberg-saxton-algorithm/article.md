# Gerchberg-Saxton Algorithm

Also called: Iterative Phase Optimization, iterative phase retrieval

## Overview
The Gerchberg-Saxton algorithm is an iterative optimization method used to compute phase-only holograms by determining object point phase values that cause interference resulting in constant amplitude across the hologram plane [S1]. It solves the problem of phase-only display constraints where the eye does not detect image phase, allowing the algorithm to treat object phases as free variables to satisfy amplitude constraints at the hologram [S1]. It differs from non-iterative methods by explicitly enforcing amplitude constraints through back-and-forth propagation, though it is computationally more expensive and prone to high-frequency noise compared to direct calculation methods [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Target image intensity distribution (implicitly defining the required amplitude at the hologram plane, typically constant or uniform), object point positions, and wavelength.
- **Output:** Optimized phase values $\phi_o$ for each object point $o$ [S1].

**Data Structures:**
- Complex field representation at the hologram plane: $H(x,y) = A_H(x,y) e^{i \phi_H(x,y)}$.
- Complex field representation at the object plane: $O(x,y) = A_O(x,y) e^{i \phi_O(x,y)}$.
- The algorithm treats $\phi_o$ (phase at object points) as the primary free variables [S1].

**Numbered Steps:**
1. **Initialization:** Initialize the phase values $\phi_o$ for each object point. A common initialization is random phase or zero phase [G]. Set the amplitude at the hologram plane $A_H$ to the desired constant value (or target amplitude distribution) [S1].
2. **Forward Propagation:** Propagate the light field from the object plane to the hologram plane using a wave propagation model (e.g., Fresnel or Fraunhofer diffraction) [S1]. This yields a complex field $H_{calc}$ at the hologram plane.
3. **Amplitude Constraint Enforcement (Hologram Plane):** Replace the amplitude of the calculated field $H_{calc}$ with the target amplitude $A_H$, while preserving the calculated phase $\phi_H$. The new field is $H_{new} = A_H e^{i \phi_H(H_{calc})}$ [S1].
4. **Backward Propagation:** Propagate the constrained field $H_{new}$ back from the hologram plane to the object plane [S1]. This yields a complex field $O_{calc}$ at the object plane.
5. **Amplitude Constraint Enforcement (Object Plane):** In the standard Gerchberg-Saxton formulation for holography, the amplitude at the object plane is often not strictly constrained to a specific value but the phase is updated. However, based on the source description, the method focuses on finding $\phi_o$ such that the resulting hologram amplitude is constant. The source implies the iteration continues by using the phase from the object plane result to update the object field for the next forward pass. Specifically, the phase $\phi_O(O_{calc})$ is used to update the object field $O_{new} = A_O e^{i \phi_O(O_{calc})}$, where $A_O$ might be the initial amplitude or left as is, but the key is updating the phase to drive the next iteration [S1]. *Note: The source emphasizes that $\phi_o$ are the free variables. The standard GS algorithm enforces constraints on both planes. In holography, the object plane constraint is often the target image intensity, but here the goal is constant hologram amplitude. The source states "light is propagated back and forth... while enforcing amplitude constraints". We enforce the constant amplitude at the hologram plane. At the object plane, we typically enforce the target image amplitude if it were an image reconstruction problem, but here the "object" is the source points. The source says "find values $\phi_o$ that cause the object points to interfere such that the required amplitude values of H are constant". This implies the object plane amplitude is fixed by the source definition, and only phase is optimized. Thus, step 5 effectively just takes the phase from the backward propagated field to update the object phase for the next iteration.*
6. **Convergence Check:** Repeat steps 2-5 for a specified number of iterations. The typical number of iterations is 10-100 [S1]. Stop if the error metric (e.g., difference between calculated hologram amplitude and target amplitude) falls below a threshold or the iteration limit is reached.

**Formulas:**
- Forward Propagation: $H_{calc} = \mathcal{P}_{obj \to holo}(O_{current})$
- Hologram Constraint: $H_{new} = A_{target} \cdot \frac{H_{calc}}{|H_{calc}|}$
- Backward Propagation: $O_{calc} = \mathcal{P}_{holo \to obj}(H_{new})$
- Object Phase Update: $\phi_o^{(k+1)} = \arg(O_{calc})$

**Parameters:**
- Number of iterations: Typically 10-100 [S1].

**Edge Cases and Pitfalls:**
- **High-frequency noise/speckle:** Iteratively optimized holograms tend to include undesirable high-frequency noise or speckle [S1].
- **Computational expense:** The method is computationally expensive due to the many iterations required [S1].
- **Local minima:** Like many iterative optimization methods, it may converge to local minima, though the source does not explicitly detail this pitfall for GS specifically, it is a general characteristic of such methods [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gerchberg_saxton_hologram(object_points, target_amplitude_holo, num_iterations=50):
    """
    Computes phase-only hologram using Gerchberg-Saxton algorithm.
    
    Args:
        object_points: List of object point positions and initial amplitudes.
        target_amplitude_holo: Constant amplitude value desired at hologram plane.
        num_iterations: Number of back-and-forth propagation steps.
        
    Returns:
        Optimized phase values for each object point.
    """
    # Initialize object field phases randomly or to zero [G]
    object_phases = np.random.uniform(0, 2 * np.pi, len(object_points))
    
    for iteration in range(num_iterations):
        # Construct object field with current phases
        # Object field is a superposition of point sources
        object_field = np.zeros((H, W), dtype=complex)
        for i, point in enumerate(object_points):
            # Calculate contribution of point i to the hologram plane
            # This involves propagating a point source from object to hologram
            # Using a propagation operator P
            contribution = propagate_point_source(point, object_phases[i])
            object_field += contribution
            
        # Forward propagation is implicit in the superposition if we define
        # the field at the hologram plane directly. However, GS usually
        # works with fields on grids. Let's assume a grid-based representation.
        
        # 1. Forward Propagation: Object Plane -> Hologram Plane
        # If object points are discrete, we sum their propagated fields.
        # Let H_calc be the complex field at the hologram plane.
        H_calc = np.zeros((H_holo, W_holo), dtype=complex)
        for i, point in enumerate(object_points):
            # Propagate point source i with phase phi_o[i]
            # This is a diffraction integral or FFT-based propagation
            H_calc += propagate_object_point_to_hologram(point, object_phases[i])
            
        # 2. Enforce Amplitude Constraint at Hologram Plane
        # Replace amplitude with target, keep phase
        H_new = target_amplitude_holo * np.exp(1j * np.angle(H_calc))
        
        # 3. Backward Propagation: Hologram Plane -> Object Plane
        # Propagate H_new back to object plane
        O_calc = propagate_hologram_to_object_plane(H_new)
        
        # 4. Update Object Phases
        # Extract phase from the backward propagated field at object point locations
        for i, point in enumerate(object_points):
            # Get the complex value at the location of object point i
            # Note: In a grid, this is sampling. In a point-source model,
            # we might need to invert the propagation or sample the field.
            # The source says "find values phi_o".
            # We take the phase of the field at the object point location.
            phase_at_point = np.angle(O_calc[point.y, point.x])
            object_phases[i] = phase_at_point
            
    return object_phases
```

## Subtopics

### [Camera-Calibrated Wave Propagation Model Training](12905-camera-calibrated-wave-propagation-model-training.md)
A method that trains a wave propagation model calibrated to a specific camera, differing from GS which uses a fixed physical propagation model. Choose this when hardware-specific aberrations need to be corrected.

## Operating characteristics
- **Time Complexity:** $O(I \cdot N \log N)$ or $O(I \cdot N^2)$ depending on the propagation implementation, where $I$ is the number of iterations (10-100) and $N$ is the number of pixels/points [S1].
- **Memory Complexity:** $O(N)$ for storing the complex fields at each plane [G].
- **Measured Performance:** Not given in the sources.
- **Strengths:** It is the most common method for computing phase-only holograms and can produce acceptable results for simple text and symbology [S1].
- **Limitations:** Computationally expensive due to many iterations; tends to produce high-frequency noise or speckle; not a practical general solution for complex scenes compared to other methods [S1].
- **When to use:** When phase-only holograms are required and computational resources allow for iterative optimization, or for simple targets like text [S1].
- **When not to use:** For complex scenes requiring high fidelity without speckle, or when real-time performance is critical [S1].

## Background
The Gerchberg-Saxton algorithm was introduced by Gerchberg and Saxton in 1972 [S1]. It is widely used in holography to compute phase-only holograms by iteratively optimizing object point phases to satisfy amplitude constraints at the hologram plane [S1]. Modern variations add error compensation and "do not care" regions to improve performance [S1].

## Sources
[S1] Holographic near-eye displays for virtual and augmented reality, https://doi.org/10.1145/3072959.3073624, fetched 2026-10-30
