# Multiresolution Atlas Buffer (MAB)

Also called: VR DVR acceleration wrapper, bracket approach for VR raycasting, gaze-dependent resolution DVR

## Overview
The Multiresolution Atlas Buffer (MAB) is an acceleration wrapper for Direct Volume Rendering (DVR) in Virtual Reality (VR) that reduces ray density adaptively based on peripheral vision and Head-Mounted Display (HMD) lens distortion, then reconstructs the full-resolution image via compositing [S1]. Unlike sibling methods that modify the ray traversal itself or use high-speed eye-tracking for foveated rendering, MAB operates as a "bracket" around existing raycasters, decoupling the adaptive sampling logic from the volume traversal algorithm [S1]. It is specifically designed to guarantee stable high refresh rates (90 Hz or higher) on high-resolution HMDs to prevent cyber-sickness, leveraging the fixed nature of HMD lenses where the highest resolution is naturally confined to a slight central angle [S1].

## How it works
The MAB method functions as a two-stage wrapper injected into an existing DVR pipeline: an "opening brace" that modifies initial ray construction by adaptively reducing ray density, and a "closing brace" that performs compositing to reconstruct the final image from the subsampled results [S1].

### Inputs and Outputs
*   **Inputs:**
    *   The current frame's screen-space coordinates $(u, v)$ of the target framebuffer.
    *   The distance of each fragment from the center of gaze $d_{gaze}$ [S1].
    *   The current system frame-timing metrics [S1].
    *   The set of pre-computed MAB regions and their resolution levels [S1].
*   **Outputs:**
    *   A set of adaptive ray coordinates sent to the raycaster (Opening Brace).
    *   A final composited framebuffer image submitted to the HMD (Closing Brace) [S1].

### Data Structures
**Multiresolution Atlas Buffer (MAB):**
The MAB is a buffer partitioned into multiple regions. Each region corresponds to the entire frame buffer at a specific resolution level [S1].
*   **Layout:** The regions are ordered to minimize the overall framebuffer size [S1].
*   **Resolution Levels:** The resolution scaling values between subsequent levels are arbitrary and not necessarily a power of two [S1].
*   **Discarding:** Black areas (empty space) in each level are discarded before ray traversal to save computation [S1].

### Mapping Functions
A bidirectional mapping between the framebuffer space and the MAB space is mandatory [S1]. Two functions compute this mapping during the compositing stage:
1.  **Blending Factor Function ($f_a$):** Takes the distance to the center of gaze $d_{gaze}$ (derived from screen-space coordinates) and outputs a set of blending factors for the MAB regions [S1].
2.  **Coordinate Mapping Function ($f_b$):** Takes the screen-space coordinates of a framebuffer fragment and outputs a set of corresponding coordinates in the MAB regions [S1].

### Algorithm Steps

**Phase 1: Opening Brace (Ray Construction)**
1.  **Metric Evaluation:** Evaluate the system's frame-timing from the previous frame [S1].
2.  **Parameter Adjustment:** Adjust rendering parameters (specifically the resolution levels and ray density) for the next subsampling stage to maintain a consistent refresh rate [S1].
3.  **Ray Generation:** Generate primary rays based on the adaptive resolution. The spatial ray frequency is reduced in peripheral regions corresponding to lower-resolution MAB levels [S1].
4.  **Ray Traversal:** Feed the reduced set of rays to the existing raycaster. The raycaster traverses the volume and stores results into the corresponding regions of the MAB [S1].

**Phase 2: Closing Brace (Compositing)**
1.  **Full-Screen Primitive:** Render a full-screen primitive covering the entire output framebuffer [S1].
2.  **Fragment Processing:** For each generated fragment in the output framebuffer:
    a.  Calculate the distance to the center of gaze $d_{gaze}$ based on the fragment's screen-space coordinates [S1].
    b.  Evaluate $f_a(d_{gaze})$ to obtain a set of blending factors $\{w_i\}$ for the MAB regions [S1].
    c.  Evaluate $f_b(u, v)$ to obtain a set of corresponding coordinates $\{(u_i, v_i)\}$ in the MAB regions [S1].
    d.  **Lookup and Blend:** For each nonzero blending factor $w_i$:
        i.  Perform a lookup into the MAB at coordinates $(u_i, v_i)$ to obtain the color value $C_i$ [S1].
        ii. Accumulate the final color $C_{final}$ using the blending factors:
        $$ C_{final} = \frac{\sum w_i C_i}{\sum w_i} $$
        *(Note: The source describes blending into a single color but does not specify the exact normalization formula; standard weighted averaging is implied by "blending factors" [S1].)*
3.  **Submission:** Submit the final composited framebuffer to the HMD [S1].

### Parameters
*   **HMD Update Rate:** At least 90 Hz, with a tendency toward 120 Hz or more [S1].
*   **HMD Resolution:** Around 2880 x 1600 pixels [S1].
*   **Resolution Scaling:** Arbitrary ratios between levels; not restricted to powers of two [S1].
*   **Gaze Center:** The center of gaze is assumed to be fixed relative to the HMD lens geometry for the purpose of this method, as rapid eye movements are not tracked [S1].

### Edge Cases and Pitfalls
*   **Eye Tracking:** The method does not consider rapid eye movements because HMD lenses are fixed and provide only a slight angle with the highest projected resolution [S1]. Implementers must not assume dynamic gaze tracking hardware is available or used.
*   **Resolution Ratios:** Unlike Mip-Maps, the ratio between levels is not fixed. Implementers must handle arbitrary scaling factors in $f_a$ and $f_b$ [S1].
*   **Frame Timing:** Failure to evaluate frame-timings and adjust parameters for the next subsampling stage may result in unstable refresh rates, leading to cyber-sickness [S1].

## Operating characteristics
*   **Performance:** The method significantly improves rendering performance in VR setups by reducing the number of primary rays traversed, making DVR feasible on hardware that might otherwise struggle with high-resolution, high-refresh-rate demands [S1].
*   **Complexity:** The computational overhead of the compositing stage (evaluating $f_a$ and $f_b$ and blending) is added to the ray traversal cost. However, the reduction in ray traversal volume typically outweighs this overhead [S1].
*   **Memory:** The MAB requires storing multiple resolution levels of the framebuffer. The layout is optimized to minimize the overall framebuffer size [S1].
*   **Strengths:**
    *   **Compatibility:** Works as a wrapper around practically any ray traversal mechanism, including recent ray-guided volume rendering systems [S1].
    *   **VR Optimization:** Specifically tailored to HMD characteristics (lens distortion, fixed gaze angle) and human visual system (HVS) properties [S1].
    *   **Stability:** Guarantees stable high refresh rates to avoid cyber-sickness [S1].
*   **Limitations:**
    *   **No Eye Tracking:** Does not support dynamic foveated rendering based on rapid eye movements [S1].
    *   **Blending Artifacts:** The quality of the final image depends on the smoothness of the blending factors and coordinate mappings; abrupt changes in resolution levels may cause visible seams if not handled carefully [S1].
*   **When to Use:** VR setups with HMDs requiring stable high refresh rates (90 Hz+) and high resolutions (e.g., 2880 x 1600) where DVR is performance-bound [S1].
*   **When Not to Use:** Desktop rendering without HMDs, or scenarios where dynamic eye-tracking is available and preferred over the fixed-gaze approximation [S1].

## Background
The Multiresolution Atlas Buffer (MAB) was introduced by Jens Krüger and Andre Waschk in the WSCG 2022 Proceedings (CSRN 3201) [S1]. It builds upon the Foveated Adaptive Volume Rendering (FAVR) approach by Waschk and Krüger (2020) and earlier gaze-dependent rendering works [S1]. The method addresses the challenge of rendering high-resolution volume data in VR, where the close proximity of the display and high refresh rate requirements make standard DVR computationally expensive [S1]. By leveraging the nonuniform resolution distribution of the human visual system and the fixed lens distortion of HMDs, MAB provides a practical acceleration technique that requires minimal changes to existing raycasting implementations [S1].

## Sources
[S1] Embracing Raycasting for Virtual Reality, https://doi.org/10.24132/csrn.3201.16, fetched 2026-10-29
