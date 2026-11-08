# MR-TF Approach

Also called: interactive ROI-based DVR in MR-HMDs, intuitive and semi-automated MR-TF approach

## Overview
The MR-TF approach is an interactive method for Direct Volume Rendering (DVR) in Mixed Reality Head-Mounted Displays (MR-HMDs) that utilizes biometric interfaces for Region of Interest (ROI) selection and optimizes Transfer Function (TF) parameters to match a target visibility, weighted by color contrast with physical backgrounds [S1]. It distinguishes itself from other optimization methods by integrating real-time biometric input (eye-gaze and finger pointing) for seed selection and employing a lightweight Nelder-Mead optimization on low-spec hardware, whereas siblings like [VWS-optimized Transfer Function Optimization](7420-vws-optimized-transfer-function-optimization.md) or [AI-Based Transfer Function Peak Prediction](2921-ai-based-transfer-function-peak-prediction.md) rely on different computational strategies or offline training [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** 3D volume data (intensity values), MR-HMD biometric data (eye convergence point, head center, finger pointing vector), user-defined target visibility ($V_T$), and ROI threshold level.
*   **Outputs:** Optimized Transfer Function parameters ($\theta$) and the resulting DVR image.

**Data Structures**
*   **Volume Data:** 3D grid of scalar intensity values.
*   **Transfer Function:** A set of tent-shaped components mapping intensity to opacity. Each component is defined by its peak intensity value (optimized) and fixed base width.
*   **ROI Mask:** Binary or probabilistic mask identifying voxels belonging to the selected region.

**Numbered Steps**

1.  **Generate Gazing Plane:** Cast a line along the y-axis connecting the convergence point of both eyes and the center point of the MR-HMD to define the gazing plane [S1].
2.  **Determine 3D Intersection Point:** Calculate the intersection between the gazing plane and the pointing vector of the user's finger [S1].
3.  **Validate and Adjust Point:** Highlight the 3D intersection point with a yellow sphere for validation. Allow manual adjustment of the point in 5-mm units along each axis via a control panel if precise placement is required [S1].
4.  **Segment ROI via Region Growing:** Use the validated 3D point as the seed for a region growing algorithm [S1].
    *   Retrieve the intensity value of the seed voxel.
    *   Iteratively propagate to adjacent voxels.
    *   Include a neighbor voxel if its intensity falls within a threshold range of the seed voxel's intensity.
    *   The threshold range is determined by a user-selected level $L$:
        $$ \text{Threshold} = L \times 30 $$
        where Level 1 corresponds to $\pm 30$ on the intensity value, increasing by 30 steps per level up to a maximum level of 10 [S1].
5.  **Initialize Transfer Function:** Formulate an initial 1D intensity-based TF as a set of tent-shaped components. The bottom side of the tent remains unchanged; only the peak of each component is subject to optimization [S1].
6.  **Compute Initial Visibility:** Calculate the initial visibility $V_C$ of the ROI in the current DVR rendering.
7.  **Optimize TF Parameters:** Iteratively optimize the opacity parameters $\theta$ (specifically the peaks of the tent components) to minimize the loss function $F(\theta)$ using the Nelder-Mead simplex method [S1].

**Formulas**

The loss function $F(\theta)$ is a weighted sum of visibility tolerance and TF movement tolerance:

$$ F(\theta) = \omega F_T(\theta) + (1 - \omega) F_M(\theta) $$

Where:
*   $\theta$: Set of opacity parameters (peaks of tent-shaped components) [S1].
*   $\omega$: Weight for the visibility tolerance [S1].
*   $F_T(\theta)$: Visibility tolerance, calculated as the square error between the user-assigned target visibility $V_T$ and the computed visibility $V_C$:
    $$ F_T(\theta) = (V_T - V_C)^2 $$
    *Note: The source text cuts off at "The tolerance of TF movements w...", so the exact formula for $F_M(\theta)$ is not provided in the available excerpts [S1].*
*   $V_T$: Target visibility, expressed as a multiple of the initial visibility of the ROI [S1].
*   $V_C$: Computed visibility of the ROI in the current rendering [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mr_tf_approach(volume_data, eye_convergence, head_center, finger_vector, target_visibility_multiple, threshold_level):
    """
    Implements the MR-TF approach for interactive ROI-based DVR.
    
    Args:
        volume_data: 3D array of intensity values.
        eye_convergence: 3D vector representing the convergence point of both eyes.
        head_center: 3D vector representing the center point of the MR-HMD.
        finger_vector: 3D vector representing the pointing direction of the finger.
        target_visibility_multiple: Float, multiplier for initial ROI visibility to set V_T.
        threshold_level: Integer, 1 to 10, determines the region growing threshold.
        
    Returns:
        optimized_tf_params: Optimized peak values for the tent-shaped TF components.
    """
    
    # Step 1: Generate Gazing Plane
    # Line along y-axis connecting eye convergence and head center
    # Note: The source describes casting a line along the y-axis connecting these points.
    # Implementation detail: This defines a plane or line for intersection.
    gazing_line_direction = normalize(eye_convergence - head_center)
    # Assuming the "y-axis" description implies a specific coordinate system alignment
    # or a projection. For intersection, we need the plane equation or line intersection.
    # Based on text: "casting, along the y-axis, a line that connects..."
    # This phrasing is slightly ambiguous. It likely means the gazing plane is defined
    # by the eye convergence and head center, and we intersect the finger ray with it.
    # Let's assume standard ray-plane intersection where the plane is defined by the
    # eye convergence and head center and the viewing direction.
    # However, the text says "casting... a line...". Let's assume the intersection
    # is between the finger ray and the plane defined by the eye convergence and head center.
    
    # Simplified interpretation for implementation:
    # 1. Define plane normal from eye convergence and head center? 
    # Or is the "gazing plane" the plane containing the eye convergence, head center, and forward vector?
    # The text says: "The gazing plane of the eyes was generated by casting, along the y-axis, 
    # a line that connects the convergence point of both eyes and the center point of the MR-HMD."
    # This implies the line itself is the generator. Let's assume the intersection point P is found
    # by intersecting the finger ray with the plane perpendicular to the gazing direction?
    # Actually, "casting a line... to generate a gazing plane" suggests the line is part of the plane definition.
    # Given the ambiguity, we calculate the intersection of the finger ray with the plane 
    # defined by the eye convergence point and the head center point and the viewing direction.
    
    # Let's stick to the explicit steps:
    # 1. Get 3D intersection point between plane (from eye-gaze) and finger pointing vector.
    intersection_point = compute_intersection(finger_vector, eye_convergence, head_center)
    
    # Step 2: Validate and Adjust Point
    # User validates with yellow sphere. Manual adjustment in 5-mm units allowed.
    # For automated implementation, we assume the user accepts the point or adjusts it.
    # Here we assume the point is finalized.
    seed_point = intersection_point
    
    # Step 3: Segment ROI
    threshold = threshold_level * 30
    roi_mask = region_growing(volume_data, seed_point, threshold)
    
    # Step 4: Initialize TF
    # Tent-shaped components. Initial peaks are set based on ROI intensity range.
    initial_tf_peaks = initialize_tent_peaks(volume_data, roi_mask)
    
    # Step 5: Compute Initial Visibility
    initial_visibility = compute_visibility(volume_data, initial_tf_peaks, roi_mask)
    
    # Step 6: Set Target Visibility
    target_visibility = initial_visibility * target_visibility_multiple
    
    # Step 7: Optimize TF Parameters
    # Minimize F(theta) = w * (V_T - V_C)^2 + (1-w) * F_M(theta)
    # Using Nelder-Mead simplex method.
    # Note: F_M(theta) formula is not provided in sources [S1].
    # We assume a placeholder for F_M or that w=1 if F_M is ignored, but the source implies both exist.
    # [G] Since F_M is not defined, we assume a standard regularization term or that the source
    # implies minimizing visibility error primarily. However, to be faithful to the source,
    # we must acknowledge the term exists. We will define a dummy F_M for code completeness
    # but note it is undefined in sources.
    
    def loss_function(theta, w=0.5):
        current_visibility = compute_visibility(volume_data, theta, roi_mask)
        ft = (target_visibility - current_visibility) ** 2
        # [G] F_M is not defined in the provided text. Assuming a small constant or zero for implementation.
        # In practice, this might be the change in TF parameters from the previous iteration.
        fm = 0.0 
        return w * ft + (1 - w) * fm

    optimized_tf_peaks = nelder_mead_minimize(loss_function, initial_tf_peaks)
    
    return optimized_tf_peaks

def compute_intersection(finger_vector, eye_convergence, head_center):
    """
    Computes the 3D intersection point.
    The source states: "3D intersection point between a plane generated from eye-gazing 
    and the pointing vector of the finger".
    The plane is generated by "casting, along the y-axis, a line that connects the 
    convergence point of both eyes and the center point of the MR-HMD".
    This implies the plane contains the line segment (eye_convergence, head_center) 
    and is oriented along the y-axis? Or the line IS the y-axis projection?
    Given the ambiguity, we assume the plane is defined by the eye convergence, 
    head center, and the forward viewing direction of the HMD.
    """
    # [G] Standard ray-plane intersection.
    # Plane defined by point P0 (eye_convergence) and normal N.
    # If the line connects eye_convergence and head_center, and is cast "along the y-axis",
    # it might mean the plane is perpendicular to the y-axis? No, "casting... a line... to generate a plane".
    # Let's assume the plane is the one containing the eye convergence, head center, and the finger ray's origin?
    # No, the finger ray intersects the plane.
    # Most likely: The plane is the sagittal plane or similar defined by the eyes and head center.
    # We will return a placeholder intersection.
    return eye_convergence # Placeholder

def region_growing(volume_data, seed_point, threshold):
    """
    Standard region growing algorithm.
    """
    seed_intensity = volume_data[seed_point]
    queue = [seed_point]
    visited = set()
    roi_mask = np.zeros_like(volume_data, dtype=bool)
    
    while queue:
        current = queue.pop(0)
        if current in visited:
            continue
        visited.add(current)
        roi_mask[current] = True
        
        current_intensity = volume_data[current]
        if abs(current_intensity - seed_intensity) <= threshold:
            neighbors = get_adjacent_voxels(current)
            for neighbor in neighbors:
                if neighbor not in visited:
                    queue.append(neighbor)
    return roi_mask

def compute_visibility(volume_data, tf_peaks, roi_mask):
    """
    Computes the visibility of the ROI given the TF parameters.
    """
    # [G] Visibility is typically the sum of opacities of voxels in the ROI 
    # or the fraction of ROI voxels with opacity > 0.
    # We assume a simple metric: sum of opacities.
    visibility = 0.0
    for voxel in roi_mask.nonzero():
        intensity = volume_data[voxel]
        opacity = evaluate_tf(intensity, tf_peaks)
        visibility += opacity
    return visibility

def evaluate_tf(intensity, tf_peaks):
    """
    Evaluates the tent-shaped TF.
    """
    # [G] Tent shape: linear rise to peak, linear fall.
    # Assuming peaks define the center of the tent.
    # This is a simplified implementation.
    opacity = 0.0
    for peak in tf_peaks:
        # Tent function centered at peak
        # Width is fixed. Let's assume width W.
        # opacity = max(0, 1 - abs(intensity - peak) / (W/2))
        # Since W is not given, we assume a standard width or that the peak
        # defines the max opacity and the base is fixed.
        pass
    return opacity
```

**Parameters**
*   **Threshold Level:** Integer from 1 to 10. Level 1 corresponds to a threshold range of $\pm 30$ on the intensity value of the seed voxel. Each subsequent level increases the threshold by 30 [S1].
*   **Adjustment Step:** 5-mm units for manual adjustment of the 3D intersection point [S1].
*   **Weight $\omega$:** Weight for visibility tolerance in the loss function. Value not specified in sources [S1].

**Edge Cases and Pitfalls**
*   **Precision:** Manual adjustment is limited to 5-mm units, which may be insufficient for high-resolution data [S1].
*   **Hardware:** The method is designed for low-spec commercial MR-HMDs, relying on simple region growing and Nelder-Mead to avoid heavy computational demands [S1].
*   **Ambiguity:** The exact formula for $F_M(\theta)$ (tolerance of TF movements) is cut off in the source [S1].
*   **Plane Definition:** The geometric definition of the "gazing plane" is described as "casting, along the y-axis, a line that connects the convergence point of both eyes and the center point of the MR-HMD," which is geometrically ambiguous for implementation without further clarification [S1].

## Subtopics

### Interactive ROI Selection
Uses biometric interfaces (eye-gaze and finger pointing) for seed selection, setting it apart from manual mouse-based selection in [VWS-optimized Transfer Function Optimization](7420-vws-optimized-transfer-function-optimization.md). Choose when immersive, hands-free interaction is required.

### Tent-Shaped TF
Uses tent-shaped components for TF parameterization, allowing semi-transparent exhibition of multiple ROIs. This differs from the lookup table optimization in [Lookup TF Optimization](6791-lookup-tf-optimization.md). Choose when smooth, iso-surface-like rendering is desired.

### Nelder-Mead Optimization
Uses derivative-free optimization suitable for low-spec hardware. This contrasts with gradient-based methods or AI-based predictions in [AI-Based Transfer Function Peak Prediction](2921-ai-based-transfer-function-peak-prediction.md). Choose when computational resources are limited.

## Operating characteristics
*   **Time Complexity:** Depends on the region growing algorithm and the number of Nelder-Mead iterations. Region growing is generally linear with respect to the number of voxels in the ROI. Nelder-Mead is iterative but lightweight [S1].
*   **Memory Complexity:** Low, as it relies on simple data structures (volume data, ROI mask, TF parameters) and does not require large models or complex grids like [Voxel-based Transfer Function Optimization](3259-voxel-based-transfer-function-optimization.md) [S1].
*   **Performance:** Designed for on-device implementation on low-spec commercial MR-HMDs [S1].
*   **Strengths:** Intuitive user interaction, robust to noise, minimal computational demands [S1].
*   **Limitations:** Limited to intensity-based 1D TFs, manual adjustment precision is low (5-mm), and the method's reliance on biometric interfaces requires compatible MR-HMD hardware [S1].
*   **When to Use:** Interactive ROI-based DVR in Mixed Reality environments where hardware resources are constrained.
*   **When Not to Use:** When high-precision manual adjustment is required, or when advanced segmentation algorithms are needed (though the method can be adapted to use remote computing for such cases) [S1].

## Background
The MR-TF approach was introduced in 2025 by the authors of the paper "An intuitive and semi-automated transfer function design for interactive region of interest-based direct volume rendering in mixed reality head mounted devices" [S1]. It builds on region growing algorithms (Monga 1987) and tent-shaped TF parameterization (Jung et al. 2018) [S1]. The method is used in medical visualization contexts, such as attenuating occluding dentin and crown regions in dental imaging [S1].

## Sources
[S1] An intuitive and semi-automated transfer function design for interactive region of interest-based direct volume rendering in mixed reality head mounted devices, https://doi.org/10.1007/s10055-025-01121-4, fetched 2026-10-30
