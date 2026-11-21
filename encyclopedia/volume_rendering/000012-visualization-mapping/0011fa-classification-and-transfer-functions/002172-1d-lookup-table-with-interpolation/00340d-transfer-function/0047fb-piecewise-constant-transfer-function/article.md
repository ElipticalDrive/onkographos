# Piecewise Constant Transfer Function

Also called: Step Transfer Function, TF1, TF2

## Overview
The Piecewise Constant Transfer Function maps voxel intensity values to RGBA color and opacity using piecewise constant intervals [S1, unofficial]. It is distinguished from its parent [Transfer Function](1272-transfer-function.md) by its lack of interpolation, classifying voxels as either fully included in a material definition or excluded [G], and differs from [Transfer Function Mapping via 1D Textures](8608-transfer-function-mapping-via-1d-textures.md) by not requiring texture sampling or linear interpolation between control points [G].

## How it works
**Inputs and Outputs**
- Input: A voxel intensity value $v$ (signed short) [S1, unofficial].
- Input: A global opacity parameter $O_{skin}$ (double, range $[0, 1]$) [S1, unofficial].
- Output: A color vector $(R, G, B)$ and an opacity value $O$ [S1, unofficial].

**Data Structures**
No complex data structures are required; the function relies on hardcoded thresholds and constant color values [S1, unofficial].

**Steps**
The implementation follows a strict `if-else` chain based on the value of $v$ [S1, unofficial]. Two variants are defined: TF1 and TF2 [S1, unofficial].
1. **Evaluate Interval 1 (Skin):** Check if $-299 < v < 50$ [S1, unofficial]. If true, set $R = 1.0$, $G = 0.79$, $B = 0.6$, and $O = O_{skin}$ [S1, unofficial].
2. **Evaluate Interval 2 (Bone) - TF1 Only:** If the first condition is false, check if $v > 300$ [S1, unofficial]. If true (TF1 only), set $R = 1.0$, $G = 1.0$, $B = 1.0$, and $O = 0.8$ [S1, unofficial]. If false (TF1) or using TF2, proceed to Step 3 [S1, unofficial].
3. **Default (Background):** Set $R = 0.0$, $G = 0.0$, $B = 0.0$, and $O = 0.0$ [S1, unofficial].

**Formulas**
For TF1:
$$
\text{Color}(v) = \begin{cases} 
(1.0, 0.79, 0.6, O_{skin}) & \text{if } -299 < v < 50 \\
(1.0, 1.0, 1.0, 0.8) & \text{if } v > 300 \\
(0.0, 0.0, 0.0, 0.0) & \text{otherwise}
\end{cases}
$$ [S1, unofficial]
For TF2:
$$
\text{Color}(v) = \begin{cases} 
(1.0, 0.79, 0.6, O_{skin}) & \text{if } -299 < v < 50 \\
(0.0, 0.0, 0.0, 0.0) & \text{otherwise}
\end{cases}
$$ [S1, unofficial]

**Parameters**
- $O_{skin}$: Opacity of the skin material. Default value in source is $0.12$. Range $[0, 1]$ [S1, unofficial].
- Thresholds: $-299$, $50$, $300$. These are hardcoded integers [S1, unofficial].

**Edge Cases and Pitfalls**
- **Boundary Precision:** The intervals are open on the left and right for the skin range ($-299 < v < 50$) [S1, unofficial]. Values exactly equal to $-299$, $50$, or $300$ fall through to the default black/transparent case [S1, unofficial]. Implementers must ensure strict inequality operators are used if matching the source behavior exactly [S1, unofficial].
- **Data Type:** The source code uses `short` for voxel input [S1, unofficial]. Ensure the comparison logic handles signed integers correctly [G].
- **TF1 vs TF2:** TF2 is a subset of TF1, removing the bone classification [S1, unofficial]. The choice depends on whether the volume contains bone structures that need to be visualized as white [S1, unofficial].

**Pseudocode**
```python
def piecewise_constant_transfer_function(voxel: int, opacity_skin: float) -> tuple:
    """
    Maps a voxel intensity to RGBA using piecewise constant intervals.
    
    Args:
        voxel: The intensity value of the voxel (signed integer).
        opacity_skin: The opacity for the skin material (float 0.0-1.0).
        
    Returns:
        A tuple (R, G, B, O) where R, G, B are floats in [0, 1] and O is float in [0, 1].
    """
    # Interval 1: Skin
    if -299 < voxel < 50:
        return (1.0, 0.79, 0.6, opacity_skin)
    
    # Interval 2: Bone (TF1 variant)
    # Note: TF2 variant skips this block entirely.
    if voxel > 300:
        return (1.0, 1.0, 1.0, 0.8)
    
    # Default: Background
    return (0.0, 0.0, 0.0, 0.0)
``` [S1, unofficial]

## Subtopics

### TF1
Classifies skin and bone using hardcoded thresholds. Sets it apart from TF2 by including the bone interval. Choose when visualizing CT scans with bone structures. [TF1](1272-transfer-function.md)

### TF2
Classifies only skin using hardcoded thresholds. Sets it apart from TF1 by excluding the bone interval. Choose when focusing solely on soft tissue visualization. [TF2](1272-transfer-function.md)

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Strengths:** Not given in the sources [S1, unofficial].
- **Limitations:** Not given in the sources [S1, unofficial].
- **When to use:** Not given in the sources [S1, unofficial].
- **When not to use:** Not given in the sources [S1, unofficial].

## Background
The method originates from the `joshcodd/volume-rendering` project, an unofficial educational implementation of volume rendering techniques [S1, unofficial]. It is presented as a basic example of classification mapping, contrasting with more complex interpolation-based methods [S1, unofficial]. The specific thresholds and colors are tailored for a "visible human dataset" CT scan, distinguishing skin from background (and bone in TF1) [S1, unofficial].

## Sources
[S1] joshcodd/volume-rendering, https://github.com/joshcodd/volume-rendering, fetched 2026-10-26 (unofficial)
