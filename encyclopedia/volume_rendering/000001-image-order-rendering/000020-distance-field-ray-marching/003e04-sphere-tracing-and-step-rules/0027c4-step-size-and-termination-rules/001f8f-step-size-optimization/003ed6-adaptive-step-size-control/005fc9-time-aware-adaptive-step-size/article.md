# Time-Aware Adaptive Step Size

Also called: Ray Step Size Steering, Ray step size adaptation, Use Case 1

## Overview
Time-Aware Adaptive Step Size is a technique for controlling rendering frame rate by dynamically adjusting the ray step size based on a predicted rendering time [S1, unofficial]. It solves the problem of maintaining a target frame rate during interactive exploration, particularly when scene complexity changes due to camera movement or transfer function adjustments [S1, unofficial]. It sets itself apart from reactive methods like OLP and PID controllers by predicting rendering time proactively before rendering each frame, thereby avoiding the one-frame lag inherent in reactive approaches [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera path, transfer function $\kappa$, total duration $T$, target rendering time $t_{target}$, reference step size $\delta_{ref}$, time prediction model $ENTIRE$, and the volume dataset.
*   **Outputs:** Adapted step size $\delta_{adapt}$ for each frame, which is used by the renderer.

**Data Structures:**
*   **Time Prediction Model ($ENTIRE$):** A learned model that takes volume, pose, and transfer function as input and outputs predicted time $t_{pred}$ [S1, unofficial].
*   **Impact Function ($G(\cdot)$):** A function modeling the relative impact of ray step size on rendering time, determined experimentally by sampling rendering times of different volumes/poses and normalizing with respect to $\delta_{ref}$ [S1, unofficial].
*   **Inverse Impact Function ($G^{-1}(\cdot)$):** The inverse of $G$, used to calculate the required step size adjustment [S1, unofficial].

**Parameters:**
*   $\delta_{ref}$: Reference ray step size used for normalization and as the starting point for adaptation [S1, unofficial].
*   $t_{pred}$: The predicted rendering time for the current frame, obtained from the $ENTIRE$ model [S1, unofficial].
*   $t_{target}$: The target rendering time for the frame (e.g., 25 ms for 40 FPS) [S1, unofficial].
*   $G(\cdot)$: Function modeling relative impact of step size on time [S1, unofficial].

**Numbered Steps:**
1.  Start a timer for the total duration $T$ [S1, unofficial].
2.  While the current timer value is less than $T$:
    a.  Get the current camera pose from the path at the current timer value [S1, unofficial].
    b.  Get the current transfer function $\kappa$ at the current timer value [S1, unofficial].
    c.  Predict the rendering time $t_{pred}$ using the $ENTIRE$ model with the volume, current pose, and current transfer function [S1, unofficial].
    d.  Calculate the adapted step size $\delta_{adapt}$ using the formula:
        $$ \delta_{adapt} = \delta_{ref} G^{-1}\left(\frac{t_{target}}{t_{pred}}\right) $$
        [S1, unofficial]
    e.  Render the volume using the adapted step size $\delta_{adapt}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def time_aware_adaptive_step_size(path, transfer_function, total_duration, t_target, delta_ref, G_inv, entire_model, volume):
    """
    Steers ray step size based on predicted rendering time.
    
    Args:
        path: Function returning camera pose given time.
        transfer_function: Function returning transfer function given time.
        total_duration: Total duration T of the animation/path.
        t_target: Target rendering time (e.g., 25 ms).
        delta_ref: Reference ray step size.
        G_inv: Inverse of the impact function G.
        entire_model: The ENTIRE time prediction model.
        volume: The volume dataset.
    """
    current_time = 0.0
    while current_time < total_duration:
        # Get current camera pose and transfer function
        pose = path(current_time)
        tf = transfer_function(current_time)
        
        # Predict rendering time using the ENTIRE model
        # ENTIRE prediction time is less than 0.3 ms [S1, unofficial]
        t_pred = entire_model(volume, pose, tf)
        
        # Calculate adapted step size
        # Formula: delta_adapt = delta_ref * G_inv(t_target / t_pred) [S1, unofficial]
        ratio = t_target / t_pred
        delta_adapt = delta_ref * G_inv(ratio)
        
        # Render the volume with the adapted step size
        # renderer(volume, pose, tf, delta_adapt)
        # [Implementation of renderer is not provided in sources]
        
        # Advance time (step size in time domain, not ray step size)
        # The source implies a loop over time, but doesn't specify the time step.
        # We assume a small time increment dt for the loop.
        dt = 0.016 # Example: 60 FPS time step, adjust as needed
        current_time += dt
```

**Edge Cases and Pitfalls:**
*   **Dramatic Time Changes:** OLP and PID controllers struggle with dramatic rendering time changes induced by transfer function adjustments, whereas ENTIRE maintains stable predictions [S1, unofficial].
*   **Manual Tuning:** PID controllers require careful manual tuning of proportional, integral, and derivative settings, which differ across volumes, renderers, and hardware [S1, unofficial]. Time-Aware Adaptive Step Size avoids this by using a learned model [S1, unofficial].
*   **Reactive Lag:** Reactive methods (OLP, PID) suffer from a one-frame lag because they adjust based on past frames [S1, unofficial]. Time-Aware Adaptive Step Size predicts proactively, avoiding this lag [S1, unofficial].
*   **Misprediction:** ENTIRE may occasionally produce fluctuations due to misprediction (e.g., frames 49 and 126 in experiments) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The time prediction using ENTIRE is less than 0.3 ms [S1, unofficial]. The overall frame time is dominated by the rendering step, which is adjusted to meet $t_{target}$.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In experiments with the CUDA raycaster and Nyx dataset, ENTIRE achieved more stable frame rates compared to OLP and PID controllers, especially during dramatic rendering time changes [S1, unofficial]. The target rendering time was 25 ms (40 FPS) [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Proactive prediction avoids one-frame lag; no manual tuning required; stable frame rates during dramatic changes; high PSNR values with imperceptible degradation in image quality [S1, unofficial].
*   **Limitations:** Requires training a learned model (ENTIRE); may have occasional fluctuations due to misprediction; fundamentally different cost profile compared to reactive methods [S1, unofficial].
*   **When to use:** Interactive volume exploration where maintaining a target frame rate is essential, especially when transfer functions or camera poses change significantly [S1, unofficial].
*   **When not to use:** When a learned model cannot be trained or is unavailable; when strict real-time guarantees are needed without any potential for misprediction-induced fluctuations [S1, unofficial].

## Background
*   **Origin:** The method is part of the ENTIRE (Learning-based Volume Rendering Time Prediction) framework [S1, unofficial].
*   **History and Development:** The method was evaluated in the context of the ENTIRE paper, which compares it against OLP and PID controllers for load balancing and frame rate control [S1, unofficial].
*   **Where it is used:** Interactive volume visualization, particularly for exploring volumes with dynamic transfer functions and camera paths [S1, unofficial].

## Sources
[S1] ENTIRE: Learning-based Volume Rendering Time Prediction, https://doi.org/10.48550/arxiv.2501.12119, fetched 2026-10-26 (unofficial)
