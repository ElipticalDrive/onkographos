# Space-time representations


## Overview
Space-time representations model dynamic scenes by extending the query space of the rendering function to include time as a direct input, omitting the explicit deformation network used in warp-based methods [S1]. Unlike warp-based methods that separate geometry and appearance into a static canonical form and a dynamic transformation, space-time representations use a single unified function to directly regress dynamic properties, making them tolerant of topology changes but prone to time inconsistency [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A query tuple $q = (p, \mathbf{d}, t)$, where $p \in \mathbb{R}^3$ is the spatial position, $\mathbf{d}$ is the viewing direction, and $t$ is the time coordinate [S1].
*   **Output:** A property vector $\xi$ (e.g., color or density) [S1].

**Data Structures:**
The representation is implemented as a unified function that conditions a 3D representation with a time component [S1]. Common implementations include neural implicit models conditioned on time or sets of 3D Gaussians shared across frames with time-dependent parameters [S1]. For efficient computation, the encoding function can be an MLP or an explicit feature grid [S1]. Exact layouts and bit-packing schemes are not given in the sources.

**Numbered Steps:**
1.  **Discretize Time:** Divide the continuous time interval into equally spaced time slots [S1].
2.  **Encode Query:** Apply an encoding function to the position, direction, and time [S1]. Two variants exist: (a) the main function conditions on time directly via $\gamma(p, \mathbf{d}, t)$ [S1], or (b) the encoding function depends on time while the main network does not via $\text{enc}(p, t)$ [S1].
3.  **Compute Features:** If using discrete time slots, compute features at these slots using an MLP or explicit feature grid [S1]. For a query at time $t$ between slots $t_i$ and $t_{i+1}$, compute the feature by interpolating between the features at these neighboring slots [S1].
4.  **Regress Property:** Pass the encoded query through the network to output $\xi$ [S1].
5.  **Enforce Time Consistency:** To mitigate time inconsistency, apply flow fields that define forward and backward mappings between neighboring time steps when available [S1].

**Formulas:**
The primary formulation extends the query space:
$$f_{\theta} : (\gamma(p, \mathbf{d}, t)) \rightarrow \xi \quad (12)$$
where $p$ is position, $\mathbf{d}$ is direction, $t$ is time, $\gamma$ is an encoding function, and $\xi$ is the output property [S1].

Alternatively, the encoding function incorporates time:
$$f : (\text{enc}(p, t), [\text{enc}(\mathbf{d})]) \rightarrow \xi \quad (13)$$
where $\text{enc}$ is a function of time, rather than the main function $f$ [S1].

**Parameters:**
*   **Time Slots:** Equally spaced intervals used to discretize the time domain for feature computation [S1]. Typical values are not given in the sources.

**Pitfalls:**
*   **Time Inconsistency:** The flexibility of the representation leads to time inconsistency and makes the output dependent on the number of observations available [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def space_time_render(p, d, t, time_slots, enc_func, network):
    """
    Render a dynamic scene using a space-time representation with discrete time slots.
    
    Args:
        p: Position vector (3,)
        d: Direction vector (3,)
        t: Time scalar
        time_slots: List of equally spaced time points [t1, t2, ..., tN]
        enc_func: Encoding function that returns feature vectors for (pos, time)
        network: Neural network that takes interpolated features and direction features
        
    Returns:
        xi: Output property (e.g., color, density)
    """
    # Clamp t to the valid range
    t_clamped = max(time_slots[0], min(t, time_slots[-1]))
    
    # Find slot index i such that time_slots[i] <= t_clamped <= time_slots[i+1]
    i = 0
    while i < len(time_slots) - 1 and time_slots[i+1] < t_clamped:
        i += 1
        
    t0 = time_slots[i]
    t1 = time_slots[i+1]
    alpha = (t_clamped - t0) / (t1 - t0) if t1 != t0 else 0.0

    # Compute features at neighboring slots
    feat0 = enc_func(p, t0)
    feat1 = enc_func(p, t1)
    dir_feat = enc_func(d, 0.0)  # Direction encoding is typically time-independent

    # Interpolate features between slots
    feat_interp = (1.0 - alpha) * feat0 + alpha * feat1

    # Regress property
    xi = network(feat_interp, dir_feat)
    return xi
```

## Subtopics

### [Warp-based Dynamic Scene Representation](6886-warp-based-dynamic-scene-representation.md)
Warp-based dynamic scene representations model temporal changes by learning a temporally global 3D canonical representation and a learnable, time-dependent deformation field that warps canonical space to match observed scenes at each timestep. Unlike [Space-time representations](6885-space-time-representations.md), which extend the query space to include time directly, warp-based methods explicitly separate geometry and appearance into a static canonical form and a dynamic transformation, ensuring long-term correspondence and smooth temporal interpolation.

## Operating characteristics
*   **Time Complexity:** Depends on the encoding function and network depth [S1]. Using discrete time slots and interpolation decreases the computational burden compared to fully continuous time modeling [S1].
*   **Memory Complexity:** Can be high if storing explicit feature grids for each time slot [S1].
*   **Strengths:** Tolerant of topology changes; can model a wider range of motion than warp-based methods; simpler implementation by omitting the deformation network [S1].
*   **Limitations:** Prone to time inconsistency; performance depends on the number of observations [S1].
*   **When to Use:** When the scene undergoes significant topological changes or when a simpler, unified representation is preferred over explicit deformation fields [S1].
*   **When Not to Use:** When long-term temporal consistency is critical and the scene has a stable topology, in which case warp-based methods may be superior [S1].

## Background
Space-time representations emerged as an alternative to warp-based methods in learning-based 3D representations, aiming to simplify the modeling of dynamic scenes by directly conditioning 3D representations with a time component [S1]. Early implementations include neural implicit models conditioned on time and sets of 3D Gaussians with time-dependent parameters [S1]. The approach gained traction for its ability to handle topology changes, though it requires techniques like flow fields to enforce time consistency [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
