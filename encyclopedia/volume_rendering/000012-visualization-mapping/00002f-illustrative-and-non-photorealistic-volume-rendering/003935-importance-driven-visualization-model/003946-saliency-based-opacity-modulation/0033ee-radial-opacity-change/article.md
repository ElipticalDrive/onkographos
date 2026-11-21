# Radial Opacity Change

Also called: Angular velocity metaphor opacity, Distance-based opacity modulation

## Overview
Radial Opacity Change is an illustrative volume rendering technique that modulates voxel opacity based on the distance from a reference point (typically the object center or a user-defined focus point) to create a focus+context effect. It solves the problem of visual clutter and occlusion during user interaction (rotation, panning, zooming) by making peripheral or distant structures more transparent, thereby revealing inner details. It sets itself apart from its siblings by explicitly modeling opacity decay as a function of radial distance, inspired by the physics of viscous rotation, whereas [Spatial Importance Map via Unsharp Masking](7928-spatial-importance-map-via-unsharp-masking.md) uses spatial frequency analysis and [Sparseness-based Color and Opacity Modulation](7932-sparseness-based-color-and-opacity-modulation.md) relies on local data density.

## How it works
The method modifies the base opacity $\alpha_{base}$ of a voxel using an exponential decay function driven by a style function $S(t)$ and the Euclidean distance from the voxel to a reference center.

**Inputs:**
- Voxel position $\vec{P}_{voxel}$ in world or object space.
- Reference center position $\vec{P}_{center}$ (usually the geometric center of the volume).
- Style function $S(t)$, a scalar value derived from the interaction state (e.g., rotation speed or zoom intensity) [S1, S2, unofficial].
- Base opacity $\alpha_{base}$ (optional, often incorporated into $S(t)$ or applied multiplicatively).

**Outputs:**
- Modulated opacity $\alpha_{voxel} \in [0, 1]$.

**Data Structures:**
- No specific complex data structures are required beyond standard volume grid access. The method is computed per-voxel or per-sample during ray casting.

**Steps:**
1. Calculate the Euclidean distance $\rho$ between the current voxel position $\vec{P}_{voxel}$ and the reference center $\vec{P}_{center}$.
   $$ \rho(\vec{P}_{voxel}, \vec{P}_{center}) = \| \vec{P}_{voxel} - \vec{P}_{center} \| $$
2. Retrieve the style function value $S(t)$, which represents the intensity of the interaction (e.g., angular velocity for rotation) [S1, S2, unofficial].
3. Compute the modulated opacity using the exponential decay formula.

**Formulas:**

For standard rotation or center-based modulation [S1, S2, unofficial]:
$$ \alpha_{voxel} = e^{-S(t) \cdot \rho(\vec{P}_{voxel}, \vec{P}_{center})} $$

For zooming interaction, where the transparency gradient shifts to a user-defined focus point $\vec{P}_{focus}$ [S2, unofficial]:
$$ \alpha_{voxel} = e^{-S(t) \cdot \max(0, \rho(\vec{P}_{voxel}, \vec{P}_{center}) - \rho(\vec{P}_{voxel}, \vec{P}_{focus}))} $$

**Parameters:**
- $S(t)$: Style function. In rotation scenarios, this is proportional to the angular velocity. In zooming, it controls the strength of the focus effect [S1, S2, unofficial].
- $\rho(\vec{P}_{voxel}, \vec{P}_{center})$: Distance from voxel to center.
- $\vec{P}_{focus}$: The initial point of interaction on the object for zooming semantics [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_radial_opacity(voxel_pos, center_pos, style_function, focus_pos=None):
    """
    Computes the modulated opacity for a voxel using Radial Opacity Change.
    
    Args:
        voxel_pos (tuple): (x, y, z) coordinates of the voxel.
        center_pos (tuple): (x, y, z) coordinates of the reference center.
        style_function (float): S(t), the interaction intensity (e.g., angular velocity).
        focus_pos (tuple, optional): (x, y, z) coordinates of the focus point for zooming.
                                     If None, uses standard radial decay from center.
    
    Returns:
        float: Modulated opacity alpha in [0, 1].
    """
    # Calculate distance from voxel to center
    dx = voxel_pos[0] - center_pos[0]
    dy = voxel_pos[1] - center_pos[1]
    dz = voxel_pos[2] - center_pos[2]
    rho_center = math.sqrt(dx*dx + dy*dy + dz*dz)
    
    if focus_pos is None:
        # Standard rotation/center-based modulation
        # alpha = exp(-S(t) * rho)
        exponent = -style_function * rho_center
        alpha = math.exp(exponent)
    else:
        # Zooming interaction modulation
        # Calculate distance from voxel to focus point
        fx = voxel_pos[0] - focus_pos[0]
        fy = voxel_pos[1] - focus_pos[1]
        fz = voxel_pos[2] - focus_pos[2]
        rho_focus = math.sqrt(fx*fx + fy*fy + fz*fz)
        
        # Formula: alpha = exp(-S(t) * max(0, rho_center - rho_focus))
        # This makes voxels closer to focus (smaller rho_focus) have larger exponent magnitude
        # if they are far from center, effectively shifting transparency.
        distance_diff = rho_center - rho_focus
        if distance_diff < 0:
            distance_diff = 0
            
        exponent = -style_function * distance_diff
        alpha = math.exp(exponent)
        
    return alpha
```

**Edge Cases and Pitfalls:**
- **Precision:** If $S(t)$ is large and $\rho$ is large, the exponent becomes very negative, potentially underflowing to 0. This is generally desired (opacity becomes 0), but ensure floating-point handling does not produce NaNs.
- **Boundary Conditions:** The formula assumes $\vec{P}_{center}$ is well-defined. For non-convex or disconnected volumes, the geometric center may be outside the data, leading to unintuitive transparency gradients.
- **Zooming Logic:** The zooming formula $\max(0, \rho_{center} - \rho_{focus})$ implies that voxels closer to the focus point than the center will have 0 exponent (opacity 1) if $\rho_{focus} \ge \rho_{center}$. This preserves context far from the focus while revealing inner parts near the focus [S2, unofficial].

## Subtopics

### [Radial Opacity Change (Zooming)](14595-radial-opacity-change-zooming.md)
Radial Opacity Change (Zooming) is an illustrative volume rendering technique that reveals inner structures by modulating voxel opacity based on the distance from a user-defined focus point, rather than the distance from the volume center. Unlike its sibling [Radial Opacity Change](7140-radial-opacity-change.md), which uses angular velocity and distance from the center to create a "tunnel" effect during rotation, this method shifts the transparency gradient to follow the user's interaction point, allowing for detailed inspection of specific regions while maintaining context elsewhere [14595-radial-opacity-change-zooming.md].

### Depth Opacity-Modulation
Depth Opacity-Modulation is a technique used to ensure visibility of voxels at the back of an atomic structure by modulating opacity based on depth, preventing self-occlusion from hiding rear features [S3]. It is distinct from the radial distance-based modulation of the parent method by using depth buffer values relative to the viewpoint. It is chosen when rendering complex atomic structures where rear features are frequently occluded by front features [S3].

**Formula:**
$$ \alpha_v = \begin{cases} Z(v), & \text{if } Z(v) \le 1 \\ 1, & \text{otherwise} \end{cases} $$
where
$$ Z(v) = \alpha_s \cdot G(g_v) \cdot r^{d_v - d_{min}(s)} $$

**Parameters:**
- $\alpha_v$: Opacity of voxel $v$ [S3].
- $\alpha_s$: Seed opacity value for the atomic structure $s$, provided by a genetic algorithm [S3].
- $G(g_v)$: Monotonically-increasing gradient magnitude opacity-modulation function mapping gradient magnitudes $g_v$ to opacity-modulation factors [S3].
- $d_v$: Depth value of voxel $v$ [S3].
- $d_{min}(s)$: Minimum depth of all voxels in the atomic structure $s$ [S3].
- $r$: Depth opacity-modulation rate, controlling the influence of voxel depth in modulation [S3].

**Pitfalls:**
- Voxels with high depth values (back side) are assigned high opacity to ensure visibility even if occluded by low depth voxels (front side). This can lead to visual artifacts if the occlusion is not handled correctly by the rendering pipeline, potentially showing "ghost" structures [S3].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel/sample, as it involves basic arithmetic operations (distance calculation, exponential). It adds minimal overhead to the ray casting loop.
- **Memory Complexity:** $O(1)$ auxiliary memory per voxel, as no additional large data structures are required.
- **Performance:** The method is computationally inexpensive. The exponential function is the most expensive operation, but it is standard in graphics hardware.
- **Strengths:** Effective for focus+context visualization during interaction. It naturally reveals inner structures without complex preprocessing.
- **Limitations:** The "angular velocity" metaphor may not intuitively map to all types of user interactions (e.g., panning). The choice of center point can be arbitrary for complex shapes.
- **When to use:** When the goal is to dynamically reveal internal structures during rotation or zooming in a volume visualization.
- **When not to use:** When precise quantitative analysis of outer structures is required, as they are intentionally made transparent.

## Background
The Radial Opacity Change technique was proposed by Sikachev, Rautek, Bruckner, and Gröller in 2010 as part of a dynamic focus+context framework for volume rendering [S1, S2, unofficial]. It is inspired by the metaphor of a viscous fluid rotating around its center, where particles at the border have the highest speed and are candidates to disappear. The method was designed to aid users in exploring data efficiently and detecting unknown features during interaction [S1, unofficial]. It is often used in conjunction with DVR-MIP interpolation to enhance the visual feedback during user interaction [S1, S2, unofficial].

## Sources
[S1] Dynamic Focus + Context for Volume Rendering, https://doi.org/10.2312/pe/vmv/vmv10/331-338, fetched 2026-10-26 (unofficial)
[S2] (PDF) Dynamic Focus+Context for Volume Rendering - ResearchGate, https://www.researchgate.net/publication/220839241_Dynamic_FocusContext_for_Volume_Rendering, fetched 2026-10-27 (unofficial)
[S3] Visibility-Aware Direct Volume Rendering, https://doi.org/10.1007/s11390-011-9428-3, fetched 2026-10-26
