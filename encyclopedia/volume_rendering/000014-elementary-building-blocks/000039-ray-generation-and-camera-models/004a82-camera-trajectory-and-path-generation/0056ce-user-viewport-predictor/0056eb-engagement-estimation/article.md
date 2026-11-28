# Engagement Estimation

Also called: Engagement Estimation Timer, Viewport Predictor with Engagement Statistics

## Overview
Engagement Estimation is a module that processes user interaction data to evaluate the type, parameters, speed, and duration of engagement, potentially predicting consecutive viewpoints or regions of interest [S1, S2, unofficial]. It serves as a bridge between raw input events and the visualization system's response, specifically feeding into the User Viewport Predictor [15801-user-viewport-predictor.md] to inform future camera trajectory decisions. It is distinguished from the predictor itself by its role in quantifying the *quality* and *intent* of the current interaction (e.g., zooming vs. panning) rather than just fitting a linear regression to historical pose data.

## How it works
The module calculates engagement metrics for three primary interaction types: rotation, panning, and zooming [S2, unofficial]. It utilizes a timer to measure the duration of these interactions and outputs specific parameters relevant to the interaction type [S1, S2, unofficial].

**Inputs and Outputs:**
*   **Inputs:** Raw interaction events (rotation, panning, zooming), current time, previous interaction state.
*   **Outputs:**
    *   Engagement duration (scalar).
    *   Interaction speed (scalar).
    *   Interaction type (enum/flag).
    *   For zooming: Coordinates of the user-interaction point (the position of initial interaction on the object) [S2, unofficial].
    *   For rotation/panning: No additional semantic-dependent navigation coordinates are output by default, though this may be added [S2, unofficial].

**Data Structures:**
*   `Timer`: A scalar value tracking the elapsed time of the current interaction.
*   `LastInteractionDuration`: A scalar storing the duration of the most recently completed interaction.
*   `InteractionPoint`: A 3D vector storing the initial interaction coordinates for zooming events.

**Steps:**
1.  **Initialize Timer:** When an interaction event (rotation, panning, or zooming) begins, initialize the `Timer` to zero.
2.  **Update Timer:** During the interaction, increment the `Timer` based on the elapsed time since the start of the interaction.
3.  **Evaluate Speed:** Calculate the speed of interaction based on the change in camera parameters (e.g., rotation angle, pan distance, zoom factor) divided by the `Timer` duration [S1, unofficial].
4.  **Capture Zoom Point:** If the interaction type is zooming, record the 3D coordinates of the object point corresponding to the initial interaction event as the `InteractionPoint` [S2, unofficial].
5.  **Finalize Interaction:** When the user finishes the interaction (event ends):
    *   Store the current `Timer` value into `LastInteractionDuration`.
    *   Zero the `Timer` for the next interaction [S2, unofficial].
6.  **Incorporate Statistics:** Use the `LastInteractionDuration`, interaction speed, and type to evaluate engagement. For zooming, this includes the duration of interaction with a particular region of an object from the viewpoint [S1, unofficial].
7.  **Predict Viewpoints:** Use the evaluated engagement parameters to make predictions about consecutive viewpoints or regions of interest [S1, unofficial].

**Formulas:**
*   Engagement Duration: $D = t_{end} - t_{start}$ [S2, unofficial].
*   Interaction Speed: $V = \frac{\Delta P}{D}$, where $\Delta P$ is the magnitude of the change in interaction parameters (e.g., rotation angle, pan distance) [S1, unofficial].
*   Engagement Score (for Animation): An exponential function is used to estimate user interaction, where $S(t_{current})$ is the score at current time $t$ [S2, unofficial]. The exact formula for $S(t)$ is not provided in the sources, but it maps duration to an interval $[0, 1]$, where 0 indicates no interaction and 1 indicates very long interaction [S2, unofficial].

**Parameters:**
*   Timer resolution: Not given in the sources.
*   Exponential function parameters: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Interaction Type Ambiguity:** The module handles rotation, panning, and zooming separately. If an interaction involves multiple types simultaneously, the sources do not specify the priority or combination logic [S2, unofficial].
*   **Zoom Point Accuracy:** The `InteractionPoint` is defined as the position of *initial* interaction. If the initial interaction is ambiguous (e.g., clicking on empty space), the behavior is not specified [S2, unofficial].
*   **Timer Zeroing:** The timer is zeroed only when the interaction finishes. If the user pauses and resumes, the timer continues to accumulate unless explicitly reset by the system [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per interaction event. The calculations involve simple arithmetic operations and timer updates.
*   **Memory Complexity:** $O(1)$. The module stores a fixed number of scalar values (timer, last duration, interaction point).
*   **Strengths:** Provides immediate feedback on user intent and engagement level, allowing the visualization system to adapt (e.g., via the Animation module) in real-time [S2, unofficial].
*   **Limitations:** The prediction of consecutive viewpoints is based on simple statistics (duration, speed) and may not capture complex user intent. The lack of semantic navigation for rotation and panning limits its utility for those interaction types [S2, unofficial].
*   **When to use:** When the visualization system needs to adapt its rendering style or focus based on user engagement levels, particularly for zooming interactions where region-of-interest is critical [S1, S2, unofficial].
*   **When not to use:** When high-precision semantic navigation is required for rotation or panning, or when the user interaction model is complex and requires more sophisticated intent recognition than simple duration and speed metrics [S2, unofficial].

## Background
Engagement Estimation was introduced in the context of Dynamic Focus + Context (DFC) for volume rendering [S1, S2, unofficial]. The method was developed to enhance the interpretation of user interactions by guiding the enhancement of the visualization. It is part of a broader framework that includes an Animation module to adjust voxel styles based on engagement levels [S1, S2, unofficial]. The approach is rooted in the idea that user interaction patterns (such as zooming into a region) can be used to infer regions of interest and adjust the visualization accordingly [S1, unofficial].

## Sources
[S1] Dynamic Focus + Context for Volume Rendering, https://doi.org/10.2312/pe/vmv/vmv10/331-338, fetched 2026-10-26 (unofficial)
[S2] PDF Dynamic Focus+Contextfor Volume Rendering - GitHub Pages, https://sbruckner.github.io/assets/pdf/Sikachev-2010-DFC.pdf, fetched 2026-10-27 (unofficial)
