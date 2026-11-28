# Selective Cutting
Also called: Selective Cutting Tool
## Overview
Selective Cutting is an interaction method for volume visualization that allows users to remove volume data layer by layer, explicitly excluding internal parts from being cut [S1, unofficial]. It builds on [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md) by providing a mechanism for sequential, non-destructive removal of outer layers to reveal internal structures, distinguishing it from standard clipping which typically suppresses data behind a plane without preserving the cutaway state for further layer-by-layer exploration [S1, unofficial].
## How it works
Inputs and outputs: The input is a volumetric dataset and user interaction commands (e.g., mouse clicks or gestures) defining the cut sequence. The output is a modified visualization where specified outer layers are removed or hidden, exposing inner volumes.

Data structures: The volume data is stored as a 3D grid of voxels. A state mask or visibility buffer tracks which layers have been "cut" or removed.

Numbered steps:
1. Initialize the volume visualization with the full dataset.
2. Receive user input indicating a desire to cut a specific outer layer.
3. Identify the voxels belonging to the current outermost visible layer.
4. Mark these voxels as removed or hidden in the visibility state.
5. Re-render the volume, excluding the marked voxels.
6. Repeat steps 2-5 to expose deeper layers.

Formulas: Not given in the sources.

Parameters: Not given in the sources.

Edge cases and pitfalls: Not given in the sources.

Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def selective_cut(volume_data, user_cut_command):
    # volume_data: 3D array of voxel values
    # user_cut_command: specifies which layer to cut
    # Returns: updated visualization state
    
    # Get current visible layers
    visible_layers = get_visible_layers(volume_data)
    
    if not visible_layers:
        return "No more layers to cut"
    
    # Identify the outermost layer based on user command
    layer_to_cut = identify_layer(visible_layers, user_cut_command)
    
    # Remove voxels in this layer from visibility
    for voxel in layer_to_cut:
        voxel.visible = False
        
    # Re-render with updated visibility
    render_volume(volume_data)
```
## Subtopics

### [Hand-rail cut gesture](22303-hand-rail-cut-gesture.md)
The hand-rail cut gesture is a whole-hand interaction technique that partitions volume data along a plane defined by the orientation and movement direction of the user's hand rail. It builds on [Selective Cutting](14293-selective-cutting.md) by allowing the user to cut the volume layer by layer, excluding internal parts from being cut. It sets itself apart from standard clipping by treating the cut as a physical incision via a sliding hand motion, providing immediate visual feedback through a drawn line.

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Scaling: Not given in the sources.
Strengths and limitations: Allows for sequential exploration of internal structures without losing outer context if implemented with undo, but may be computationally expensive if re-rendering is frequent.
When to use it and when not: Use when exploring internal anatomy or structures layer by layer. Not suitable for real-time applications requiring rapid, arbitrary clipping.
## Background
Origin: The concept is described in the context of anatomical atlases and volume visualization interaction techniques [S1, unofficial].
History and development: It is mentioned as an interesting feature in approaches using cutting planes for visualizing inner parts of anatomical datasets [S1, unofficial].
Where it is used: Primarily in medical visualization and anatomical atlases for exploring internal structures [S1, unofficial].
## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
