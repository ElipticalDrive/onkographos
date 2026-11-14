# MIDA Visualization

Also called: Visible range sliders

## Overview
MIDA Visualization is a density range windowing technique that allows users to define settable ranges of visualized values, enabling selective visibility of specific structures within a volume [S1, unofficial]. It distinguishes itself from standard thresholding by supporting multiple, independent visible ranges (envelopes) for different parts of the volume, facilitating complex exploration tasks such as carving and peeling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volumetric dataset (3D grid of scalar values) and a set of user-defined visible ranges (min/max pairs) [S1, unofficial].
- Output: A rendered image where voxels falling outside the defined ranges are treated as transparent or invisible [S1, unofficial].

**Data Structures:**
- **Visible Range Envelope:** A collection of intervals $[min_i, max_i]$. Each interval defines a sub-range of density values that are rendered [S1, unofficial].
- **User Interaction State:** Tracks the current active envelope and the mode of interaction (e.g., 2D painting, slider adjustment) [S1, unofficial].

**Steps:**
1. **Initialize Ranges:** Define initial visible ranges. To hide all data, set an empty range where $min = max$ [S1, unofficial].
2. **Define Envelopes:** Users adjust the min/max values for specific regions using visible range sliders or painting tools [S1, unofficial].
3. **Render:** For each voxel, check if its density value $v$ falls within any defined interval $[min_i, max_i]$. If yes, render it; otherwise, treat it as transparent [S1, unofficial].
4. **Interact:** Users can rotate, move, and adjust parameters within the context of the selected envelope [S1, unofficial].

**Formulas:**
A voxel with density $v$ is visible if:
$$ \exists i : min_i \le v \le max_i $$
If no such $i$ exists, the voxel is invisible [S1, unofficial].

**Parameters:**
- **Visible Range Sliders:** Controls the min/max visualized values for specific regions [S1, unofficial].
- **Empty Range:** A special case where $min = max$ effectively carves away the selected region [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Fuzzy Data:** In data with low intensity variability (e.g., MRI soft tissues), partial volume effects make it difficult to isolate features [S1, unofficial].
- **Ambiguity:** Semi-translucent representations can lead to ambiguities in spatial relationships, especially when closer features clash with further ones [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_visible(density, envelopes):
    """
    Check if a voxel density is within any of the defined visible envelopes.
    
    Args:
        density (float): The density value of the voxel.
        envelopes (list of tuples): List of (min, max) tuples defining visible ranges.
        
    Returns:
        bool: True if the density is within any envelope, False otherwise.
    """
    for min_val, max_val in envelopes:
        if min_val <= density <= max_val:
            return True
    return False

def render_voxel(density, envelopes, opacity_function):
    """
    Render a single voxel based on its density and the current envelopes.
    
    Args:
        density (float): The density value of the voxel.
        envelopes (list of tuples): List of (min, max) tuples defining visible ranges.
        opacity_function (function): A function that maps density to opacity.
        
    Returns:
        float: The opacity of the voxel (0 if invisible).
    """
    if is_visible(density, envelopes):
        return opacity_function(density)
    else:
        return 0.0
```

## Subtopics

### 2D Painting Mode
A technique for visualizing fuzzy data with low intensity variability (e.g., MRI scans of soft tissues) by painting an arbitrarily shaped 2D window on the screen to provide semi-translucent spatial representations and aid in understanding spatial relationships [S1, unofficial]. It is chosen when 3D isolation is difficult due to partial volume effects and ambiguous depth cues [S1, unofficial]. Steps include painting a 2D envelope on the screen and adjusting visualization parameters within that context [S1, unofficial]. Pitfalls include difficulty in isolating features due to lack of clearly defined surfaces [S1, unofficial]. Usage is common for MRI scans of soft tissues like brain matter or fruit pulp [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Flexibility in exploring internal features through carving and peeling; intuitive interaction via painting metaphors [S1, unofficial].
- **Limitations:** Ambiguities in spatial interpretation for fuzzy data; difficulty in isolating features in 3D space [S1, unofficial].
- **When to Use:** For volumes where users need to selectively visualize specific density ranges or carve away occluding parts [S1, unofficial].
- **When Not to Use:** For data with clearly defined surfaces where simple thresholding suffices [S1, unofficial].

## Background
MIDA Visualization was introduced in the context of context-preserving volume image exploration using a 3D painting metaphor [S1, unofficial]. It builds on the concept of density range windowing to allow more complex interactions such as carving and peeling [S1, unofficial]. The method was evaluated through user studies, showing positive results in terms of ease of learning and control [S1, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
