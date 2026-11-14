# SVT Animation Interpolation

Also called: SVT Frame Interpolation

## Overview
SVT Animation Interpolation is a technique used to smooth the playback of animated Sparse Volume Textures (SVTs) by blending between two adjacent frames using a scalar alpha value [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Two adjacent SVT frame textures (current and next), a scalar interpolation value $\alpha \in [0, 1]$, and the SVT Controller instance [S1].
- **Outputs:** A single blended SVT frame texture rendered in the material graph [S1].

**Data Structures:**
The method relies on the `Animated Sparse Volume Texture Controller` exposed via the SVT Blueprint [S1]. The controller manages the internal state of the SVT, including frame indexing and timing [S1]. The material graph utilizes standard scalar parameters and texture sample parameters to interface with the controller [S1].

**Steps:**
1. **Retrieve Frames and Alpha:** Call the `Get Current Frames For Interpolation` function on the SVT Controller to retrieve the two closest frames and a scalar value $\alpha$ between 0 and 1 [S1].
2. **Setup Material Parameters:** Create two SVT texture parameters in the material graph for the retrieved frames, and one scalar parameter for the interpolation alpha [S1].
3. **Interpolate:** Use an `Interpolation` node in the Material Graph to blend between the two frame textures using the alpha value [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_svt_interpolated_frame(controller, material_graph):
    # Step 1: Retrieve the two adjacent frames and the interpolation alpha
    frame_current, frame_next, alpha = controller.get_current_frames_for_interpolation()
    
    # Step 2: Setup material parameters
    # Bind frame_current to the first SVT texture parameter
    # Bind frame_next to the second SVT texture parameter
    # Bind alpha to the scalar interpolation parameter
    
    # Step 3: Perform interpolation in the material graph
    # The Interpolation node computes the weighted average of the two frames
    blended_frame = material_graph.interpolation_node(
        a=frame_current,
        b=frame_next,
        alpha=alpha
    )
    
    return blended_frame
```

**Parameters:**
- **Alpha ($\alpha$):** A scalar value between 0 and 1, where 0 represents the current frame and 1 represents the next frame [S1].
- **Frame Rate:** Configured via the `Frame Rate` property of the controller, with a default of 24 fps [S1].

**Edge Cases and Pitfalls:**
- **Precision and Streaming:** Real-time playback performance depends heavily on resolution and sparseness; if data streaming is too slow, rendering falls back to coarser mip levels, causing visible fidelity drops [S1].

## Subtopics

### SVT Sequencer Control
SVT Sequencer Control extends the animated SVT Blueprint to allow playback control via Unreal Engine Sequencer by exposing an animation alpha variable to cinematics [S1]. It is chosen when precise, keyframe-driven animation control is required in a cinematic context, rather than real-time event-driven playback [S1].

**Steps:**
1. **Expose Variable:** In the animated SVT Blueprint, add a Fraction node and promote its A input to a variable named `Animation Alpha` [S1].
2. **Connect to Controller:** Get the Controller variable, add a `Get Duration` node, and connect `Event Tick` execution to `Get Duration` [S1].
3. **Set Time:** Add a `Set Time` node (from the Animated Sparse Volume Texture Controller category) and connect the Controller to the Target pin [S1].
4. **Calculate Time:** Multiply the `Get Duration` return value by the `Animation Alpha` variable, and connect the result to the Time input of the `Set Time` node [S1].
5. **Expose to Cinematics:** In the Details panel for the `Animation Alpha` variable, check "Expose to Cinematics" [S1].
6. **Sequencer Integration:** Add a Track to Sequencer for the Blueprint and key the `AnimationAlpha` variable in the Sequencer panel to control playback [S1].

## Operating characteristics
- **Time Complexity:** Real-time playback performance is not strictly defined by algorithmic complexity but by I/O streaming performance, depending on resolution and sparseness [S1].
- **Memory/Streaming:** If volume data takes too long to stream in, rendering falls back to coarser mip levels, causing a visible drop in fidelity [S1].
- **Optimization:** Using a smaller pixel format when importing a VDB improves streaming performance [S1].
- **Strengths:** Provides smooth temporal interpolation for animated volumes and integrates with Sequencer for cinematic control [S1].
- **Limitations:** Performance is sensitive to data size and sparseness, with fallback to lower mip levels degrading visual quality [S1].

## Background
SVT Animation Interpolation is part of the Sparse Volume Textures feature set in Unreal Engine 5.8, developed by Epic Games [S1]. It builds upon the `Animated Sparse Volume Texture` Blueprint system, which allows for real-time playback of animated VDB data [S1]. The technique is analogous to skeletal mesh animation control via Sequencer, adapting that workflow for volumetric data [S1].

## Sources
[S1] Sparse Volume Textures in Unreal Engine | Unreal Engine 5.8 ... - Epic Dev, https://dev.epicgames.com/documentation/en-us/unreal-engine/sparse-volume-textures-in-unreal-engine, fetched 2026-10-28
