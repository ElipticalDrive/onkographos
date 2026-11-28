# Calibration via Coefficient Adjustment

Also called: Aberration Calibration

## Overview
Calibration via Coefficient Adjustment is a user-in-the-loop method for correcting optical aberrations in holographic near-eye displays by empirically determining the coefficients of the holographic object point representation [S1]. It solves the problem of unknown or complex aggregate optical errors in the display and eye system without requiring explicit knowledge of the optical surfaces [S1]. It differs from automated alignment methods, such as Parallax-barrier and Lenticular Display Calibration, which use computer vision and homography to map physical components, by relying on direct human visual feedback to optimize mathematical parameters [S1, 19679-parallax-barrier-and-lenticular-display-calibration.md].

## How it works
**Inputs and Outputs:**
*   **Input:** A holographic display system capable of rendering a single object point; a user.
*   **Output:** A set of optimal coefficients $a_{jo}$ for the holographic representation function.

**Data Structures:**
The holographic representation of a single object point $o_i$ is defined by the function $f_z(\rho, \theta)_{o_i}$ [S1]. This function is parameterized by a set of coefficients $a_{jo}$ [S1]. The specific internal layout of these coefficients is not detailed, but they correspond to orthogonal geometric operations such as scale, squash, and rotate [S1].

**Steps:**
1.  **Select Object Point:** Choose a specific object point $o_i$ to calibrate [S1].
2.  **Render Test Pattern:** The display presents the single object point $o_i$ to the user, represented in the hologram as $f_z(\rho, \theta)_{o_i}$ [S1].
3.  **User Adjustment:** The user adjusts the coefficients $a_{jo}$ of $f_z(\rho, \theta)_{o_i}$ [S1].
4.  **Feedback Loop:** The user observes the resulting spot and adjusts coefficients until the most tightly focused spot is seen [S1].
5.  **Repeat:** Repeat steps 1-4 for additional object points $o_i$ across the display volume [S1].
6.  **Interpolation:** For true 3D holograms, perform calibration for a small number of focal positions (e.g., four) and interpolate the coefficients over the complete focal range [S1]. Interpolate coefficients for remaining spatial points, as aberrations vary smoothly [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calibrate_coefficients(display, user_feedback):
    """
    Calibrate holographic coefficients via user feedback.
    
    Args:
        display: Object representing the holographic display hardware.
        user_feedback: Function that returns a quality metric or allows 
                       user input to adjust coefficients.
    
    Returns:
        dict: A mapping of (object_point_id, focal_position) to 
              coefficient vectors a_jo.
    """
    # Define sparse set of object points to measure
    # For highly aberrated systems, ~15 points are sufficient [S1]
    # For low aberration, 1 point at center is sufficient [S1]
    measured_points = get_sparse_object_points(display)
    
    # Define focal positions for 3D interpolation [S1]
    focal_positions = [z1, z2, z3, z4] 
    
    calibration_data = {}
    
    for o_i in measured_points:
        for z in focal_positions:
            # Render single object point o_i at focal position z [S1]
            # Representation: f_z(rho, theta)_o_i [S1]
            
            # Initialize coefficients a_jo [G]
            current_coefficients = initialize_coefficients()
            
            # User adjusts coefficients until tightest focus is seen [S1]
            # This is an interactive loop, not automated optimization
            current_coefficients = user_feedback.adjust_until_focused(
                display, o_i, z, current_coefficients
            )
            
            calibration_data[(o_i, z)] = current_coefficients

    # Interpolate coefficients for all other points and focal positions [S1]
    # Aberrations are assumed to be smoothly varying [S1]
    full_coefficient_map = interpolate_coefficients(calibration_data)
    
    return full_coefficient_map
```

**Parameters:**
*   **Number of measured points (highly aberrated):** 15 [S1].
*   **Number of measured points (low aberration):** 1 (center of display) [S1].
*   **Number of focal positions:** 4 (example) [S1].

**Edge Cases and Pitfalls:**
*   It is impractical to measure $f_z(\rho, \theta)_{o_i}$ for all object points because there are expected to be millions [S1].
*   The method relies on the assumption that aberrations are smoothly varying [S1].

## Subtopics

### [Parallax-barrier and Lenticular Display Calibration](19679-parallax-barrier-and-lenticular-display-calibration.md)
A procedure to automatically calibrate the position of a parallax-barrier with respect to the underlying rear-projection image using a digital camera and homography. It differs from manual alignment by using a computer-guided laser-cut barrier and automated image processing to map projector sub-pixels to specific viewing angles.

## Operating characteristics
*   **Time Complexity:** The calibration process is interactive and user-dependent. The calculation of the hologram for a single point is fast, but the user adjustment loop dominates the time.
*   **Memory Complexity:** Requires storing coefficients for a sparse set of points and focal positions, then interpolating. Not given in the sources.
*   **Strengths:** Does not require knowledge of the specific optics of the display or eye; measures the aggregate effect of all components [S1]. The coefficients are applicable for any image content [S1].
*   **Limitations:** Requires user interaction; impractical for all millions of object points [S1]. Relies on smooth variation of aberrations for interpolation accuracy [S1].
*   **When to use:** When optical aberrations are significant and unknown, or when explicit optical modeling is intractable [S1].

## Background
This method is described in the context of Holographic near-eye displays for virtual and augmented reality [S1]. It addresses the complexity of the Fresnel holographic model and the need for high-speed calculation while maintaining visual quality through aberration correction [S1].

## Sources
[S1] Holographic near-eye displays for virtual and augmented reality, https://doi.org/10.1145/3072959.3073624, fetched 2026-10-30
