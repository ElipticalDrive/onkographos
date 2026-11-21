# MR-TF Approach

Also called: interactive ROI-based DVR in MR-HMDs, intuitive and semi-automated MR-TF approach

## Overview
The MR-TF approach is an interactive method for Direct Volume Rendering (DVR) in Mixed Reality Head-Mounted Displays (MR-HMDs) that utilizes biometric interfaces for Region of Interest (ROI) selection and optimizes Transfer Function (TF) parameters to match a target visibility, weighted by color contrast with physical backgrounds [S1]. It distinguishes itself from other optimization methods by integrating real-time biometric input (eye-gaze and finger pointing) for ROI definition and employing a Nelder-Mead simplex optimization on a parameterized tent-shaped TF to balance visibility tolerance against parameter stability [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume data (scalar field), MR-HMD tracking data (eye convergence, head center, finger pointing vector), user-defined target visibility $V_T$ (hyperparameter), and initial TF parameters $\theta$.
*   **Output:** Optimized TF parameters $\theta^*$ and the resulting DVR image.

**Data Structures:**
*   **Volume Data:** 3D grid of scalar intensity values.
*   **Transfer Function:** A set of tent-shaped components mapping intensity to opacity. Each component is defined by its peak intensity value and peak opacity. The base of the tent (opacity at the boundaries) remains fixed during optimization [S1].
*   **ROI Mask:** Binary mask identifying voxels belonging to the selected Region of Interest, generated via region growing.

**Steps:**

1.  **Biometric ROI Selection:**
    *   Generate a gazing plane by casting a line along the y-axis connecting the convergence point of both eyes and the center point of the MR-HMD [S1].
    *   Calculate the 3D intersection point between this gazing plane and the pointing vector of the user's finger [S1].
    *   Display a yellow sphere at the intersection point for validation. Allow manual adjustment of the 3D position in 5-mm units along each axis via a control panel if precise placement is required [S1].

2.  **ROI Segmentation:**
    *   Use the validated 3D intersection point as the seed voxel for a region growing algorithm [S1].
    *   Retrieve the intensity value $I_{seed}$ of the seed voxel.
    *   Define a threshold range $[I_{seed} - \Delta, I_{seed} + \Delta]$. The threshold level $L$ is selected by the user (e.g., via voice command). Level 1 corresponds to $\Delta = 30$ intensity units. The threshold increases by 30 steps per level up to a maximum level of 10 (i.e., $\Delta = 30 \times L$) [S1].
    *   Iteratively propagate to adjacent voxels. If a neighbor's intensity falls within the threshold range, add it to the ROI and continue propagation. Stop when no neighboring voxels fall within the range [S1].

3.  **Initial TF Setup:**
    *   Initialize the TF as a set of tent-shaped components. The intensity range representing the ROI is mapped to opacity parameters. The tent shape is chosen to depict iso-surfaces and allow semi-transparent exhibition of multiple ROIs [S1].
    *   The bottom sides of the tents (base opacity) remain unchanged during optimization. Only the peak positions and heights are optimized [S1].

4.  **Visibility Computation:**
    *   Perform DVR using the current TF parameters $\theta$ to compute the current visibility $V_C$ of the ROI [S1].
    *   $V_C$ is derived from the rendered image, comparing the ROI's appearance against the background [S1].

5.  **TF Parameter Optimization:**
    *   Minimize the loss function $F(\theta)$ using the Nelder-Mead simplex method [S1]:
        $$ F(\theta) = \omega F_T(\theta) + (1 - \omega) F_M(\theta) $$
    *   Where:
        *   $\theta$: Set of opacity parameters of the initial TF (specifically the peaks of the tent-shaped components) [S1].
        *   $\omega$: Weight for the visibility tolerance [S1].
        *   $F_T(\theta)$: Visibility tolerance loss, calculated as the squared error between the user-assigned target visibility $V_T$ and the computed visibility $V_C$:
            $$ F_T(\theta) = (V_T - V_C)^2 $$
        *   $F_M(\theta)$: Tolerance of TF movements (penalizing large changes in parameters to ensure stability) [S1].
    *   The target visibility $V_T$ is an intuitive hyperparameter expressed as a multiple of the initial visibility of the ROI, set via voice commands [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mr_tf_approach(volume, eye_convergence, head_center, finger_vector, target_visibility_multiple, omega):
    # Step 1: Biometric ROI Selection
    # Generate gazing plane line along y-axis from eye convergence to head center
    gazing_line = Line(start=eye_convergence, direction=Vector(0, 1, 0))
    
    # Calculate 3D intersection point with finger pointing vector
    # Note: Implementation details for line-line intersection depend on coordinate system
    intersection_point = intersect_lines(gazing_line, finger_vector)
    
    # Display validation sphere and allow manual adjustment in 5mm units
    # This is handled by the UI loop, returning adjusted point if modified
    validated_point = get_user_validated_point(intersection_point, step_size=5.0)
    
    # Step 2: ROI Segmentation via Region Growing
    seed_voxel = get_voxel_at(validated_point, volume)
    seed_intensity = seed_voxel.intensity
    
    # Determine threshold based on user-selected level L
    # Level 1 = +/- 30, max level 10
    threshold_delta = 30 * get_user_selected_level()
    threshold_range = (seed_intensity - threshold_delta, seed_intensity + threshold_delta)
    
    roi_mask = region_growing(volume, seed_voxel, threshold_range)
    
    # Step 3: Initial TF Setup
    # Initialize tent-shaped TF components based on ROI intensity range
    initial_tf = initialize_tent_tf(roi_mask, volume)
    theta = initial_tf.parameters
    
    # Step 4 & 5: Optimization Loop
    # Define loss function components
    def compute_visibility(tf_params, roi_mask, volume):
        # Perform DVR rendering
        rendered_image = render_dvr(volume, tf_params)
        # Compute visibility metric for ROI
        # This involves comparing ROI appearance to background
        return compute_visibility_metric(rendered_image, roi_mask)
    
    def loss_function(params):
        # Compute current visibility
        current_visibility = compute_visibility(params, roi_mask, volume)
        
        # Target visibility is multiple of initial visibility
        # Note: Initial visibility must be computed once at start
        target_visibility = target_visibility_multiple * initial_visibility
        
        # Visibility tolerance loss
        ft = (target_visibility - current_visibility) ** 2
        
        # Movement tolerance loss (penalize deviation from previous params)
        # Assuming 'prev_theta' is tracked across iterations
        fm = sum((p - prev_p)**2 for p, prev_p in zip(params, prev_theta))
        
        return omega * ft + (1 - omega) * fm
    
    # Use Nelder-Mead simplex method to minimize loss
    optimized_theta = nelder_mead_minimize(loss_function, initial_tf.parameters)
    
    return optimized_theta
```

**Edge Cases and Pitfalls:**
*   **Precision:** Manual adjustment is limited to 5-mm units [S1].
*   **Threshold Selection:** The threshold level is discrete (1 to 10), which may lead to coarse ROI segmentation if the intensity variation is high [S1].
*   **Computational Load:** The method is designed for low-spec commercial MR-HMDs, relying on simple region growing and Nelder-Mead, which may not scale well to very large volumes or complex TF shapes [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the region growing algorithm (typically linear in the number of voxels in the ROI) and the Nelder-Mead optimization iterations. Nelder-Mead is derivative-free and generally efficient for low-dimensional parameter spaces but can be slow for high dimensions [S1].
*   **Memory Complexity:** Requires storage for the volume data, ROI mask, and TF parameters. Region growing requires a queue for voxels, bounded by the ROI size [S1].
*   **Performance:** Designed for on-device implementation on low-spec MR-HMDs, suggesting real-time or near-real-time interactivity [S1].
*   **Strengths:** Intuitive user interaction via biometrics; semi-automated ROI selection; robust to noise due to region growing; simple TF parameterization [S1].
*   **Limitations:** Limited to scalar intensity-based TFs; threshold-based ROI segmentation may not capture complex structures; Nelder-Mead may converge slowly or to local minima; visibility metric computation is non-trivial and not fully detailed [S1].

## Background
The MR-TF approach was introduced in 2025 by the authors of the paper "An intuitive and semi-automated transfer function design for interactive region of interest-based direct volume rendering in mixed reality head mounted devices" [S1]. It addresses the challenge of interactive volume visualization in Mixed Reality environments, where users need to intuitively select and highlight specific regions of interest (ROIs) within complex 3D data. The method builds on previous work in biometric interaction for VR/MR and automatic TF optimization [S1].

## Sources
[S1] An intuitive and semi-automated transfer function design for interactive region of interest-based direct volume rendering in mixed reality head mounted devices, https://doi.org/10.1007/s10055-025-01121-4, fetched 2026-10-30
