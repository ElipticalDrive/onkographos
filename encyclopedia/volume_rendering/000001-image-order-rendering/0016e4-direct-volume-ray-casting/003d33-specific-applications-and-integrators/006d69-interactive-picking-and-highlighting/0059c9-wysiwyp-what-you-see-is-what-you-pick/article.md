# WYSIWYP (What You See Is What You Pick)
Also called: Visibility-Oriented Picking, WYSIWYG picking, WYSIWYP Picking, opacity-jump picking

## Overview
WYSIWYP is a volume picking technique that enables users to intuitively select 3D spatial positions in volumetric renderings based solely on the volume data, the transfer function, and the user's perception of visibility, without requiring pre-segmented data or metadata [S1]. It identifies the 3D position corresponding to a 2D screen click by finding the interval along the viewing ray with the largest jump in accumulated opacity, characterized by analyzing the second derivative of accumulated opacity [S1].

## How it works
**Inputs:**
*   Volume scalar field data $d(x)$ [S1].
*   Transfer function mapping data values to source color $C_{src}$ and source opacity $\alpha_{src}$ [S1].
*   Camera position (eye point) and screen click coordinates [S1].
*   Step size $h$ (must match DVR rendering step size for consistency) [S1].

**Outputs:**
*   A 3D position $p$ representing the picked feature [S1].

**Data Structures:**
*   Ray parameter $s$ (distance along ray) [G].
*   Accumulated opacity $\alpha_{acc}(s)$ [S1].
*   First derivative $\beta_{acc}(s) = \frac{d\alpha_{acc}}{ds}$ [S1].
*   Second derivative $\gamma_{acc}(s) = \frac{d^2\alpha_{acc}}{ds^2}$ [S1].
*   Jump interval $I = [i_0, i_{max}]$ and magnitude $j$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wysiwy_pick(camera_eye, screen_click, volume_data, transfer_function, step_size):
    # 1. Ray Definition
    # Transform screen click to world coordinates to define viewing ray
    ray_origin = screen_click
    ray_direction = normalize(screen_click - camera_eye)

    # 2. Entry Point Determination
    # Use a transparent bounding cube (proxy geometry) around the volume
    # Compute intersection of ray with AABB bounds
    bounds = volume_data.bounds
    t_min = max((bounds.min - ray_origin) / ray_direction)
    t_max = min((bounds.max - ray_origin) / ray_direction)
    entry_t = max(t_min, 0.0)
    if entry_t > t_max:
        return None  # Ray misses volume
    entry_point = ray_origin + ray_direction * entry_t

    # 3. Ray Casting and Accumulation
    alpha_acc = 0.0
    alpha_history = []
    s_current = entry_t

    # Step through the volume until exit
    while s_current <= t_max:
        pos = ray_origin + ray_direction * s_current

        # Sample data and apply transfer function
        d_val = volume_data.sample(pos)
        alpha_src, _ = transfer_function(d_val)

        # Update accumulated opacity using front-to-back compositing
        # alpha_acc^{n+1} = alpha_acc^n + (1 - alpha_acc^n) * alpha_src
        alpha_acc_new = alpha_acc + (1.0 - alpha_acc) * alpha_src
        alpha_history.append(alpha_acc_new)
        alpha_acc = alpha_acc_new

        # Advance step
        s_current += step_size

    # 4. Derivative Calculation
    # Compute first and second derivatives of alpha_acc along the ray parameter s
    beta_acc = []
    for i in range(len(alpha_history) - 1):
        beta = (alpha_history[i+1] - alpha_history[i]) / step_size
        beta_acc.append(beta)

    gamma_acc = []
    for i in range(len(beta_acc) - 1):
        gamma = (beta_acc[i+1] - beta_acc[i]) / step_size
        gamma_acc.append(gamma)

    # 5. Jump Interval Detection
    intervals = []
    i = 0
    while i < len(gamma_acc):
        # Lower bound i0: gamma_acc becomes positive after being negative or zero
        if gamma_acc[i] > 0 and (i == 0 or gamma_acc[i-1] <= 0):
            start_idx = i

            # Upper bound imax: gamma_acc becomes zero or positive after being negative
            # Detects the end of the positive gamma region
            end_idx = None
            for k in range(i, len(gamma_acc)):
                if k > 0 and gamma_acc[k-1] > 0 and gamma_acc[k] <= 0:
                    end_idx = k
                    break
            if end_idx is None:
                end_idx = len(gamma_acc)

            # Map to alpha indices
            i0 = start_idx
            imax = end_idx

            # Calculate jump magnitude j = alpha_acc(i0) - alpha_acc(imax) [S1]
            # Magnitude is taken as absolute difference since alpha_acc is monotonically increasing
            j = abs(alpha_history[imax] - alpha_history[i0])

            intervals.append({
                'i0': i0,
                'imax': imax,
                'j': j,
                'pos_i0': entry_point + ray_direction * (i0 * step_size),
                'pos_imax': entry_point + ray_direction * (imax * step_size)
            })
            i = end_idx
        else:
            i += 1

    # 6. Selection
    if not intervals:
        return None

    # Select interval with largest jump magnitude
    best = max(intervals, key=lambda x: x['j'])

    # Determine final position: front position i0 for labeling, center for slices
    picked_position = best['pos_i0']
    return picked_position
```

**Parameters:**
*   **Step Size:** Must match the step size used for DVR rendering. If different, opacity correction must be applied [S1].
*   **Sample Density:** Assumes equidistant samples. Extendable to non-equidistant samples with opacity correction [S1].

**Edge Cases and Pitfalls:**
*   **Step Size Inconsistency:** If DVR uses half the step size (twice as many samples) as the picking ray, accumulated opacity varies differently, leading to incorrect picking [S1].
*   **Plateaus:** If $\alpha_{acc}$ has plateau-like regions, $\gamma_{acc}$ is constantly zero. The boundary detection is adapted to handle this by checking for transitions from negative/zero to positive for lower bounds, and zero/positive after negative for upper bounds [S1].
*   **Noisy Data:** The method does not ensure that close positions in screen space result in close 3D locations, particularly in noisy datasets [S1].
*   **Local Illumination:** The method is limited to the standard emission-absorption model and does not currently handle local illumination applied after evaluating the transfer function [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray, $O(N)$, where $N$ is the number of steps through the volume [G].
*   **Memory Complexity:** $O(N)$ to store the history of accumulated opacity for derivative calculation [G].
*   **Measured Performance:** In a user study with 36 test cases, WYSIWYP picked positions within one standard deviation of the mean perceived position for 89% (32 of 36) of cases [S1].
*   **Strengths:** Does not require metadata or pre-segmented data; applicable to any volumetric scalar dataset and all types of transfer functions, including "foggy" images; handles cases where threshold-based picking fails (thresholds too high/low) [S1].
*   **Limitations:** Does not handle local illumination; does not guarantee spatial coherence for adjacent screen pixels in noisy data; limited to standard emission-absorption model [S1].
*   **When to Use:** When users need to pick visible structures in arbitrary volume data without metadata, or for automated slice positioning [S1].

## Background
WYSIWYP was introduced to overcome limitations of previous picking methods that required metadata, were designed for medical data only, or provided only basic techniques like first-hit or opacity-threshold [S1]. It is based on the principle that users want to examine structures made visible by DVR and suitable transfer functions [S1]. The method was implemented in the Amira visualization software and applied to abdominal MRI scans, flow simulation vorticity fields, synthetic scalar functions, and fuel injection combustion chamber simulations [S1].

## Sources
[S1] WYSIWYP: What You See Is What You Pick, https://doi.org/10.1109/tvcg.2012.292, fetched 2026-11-05
