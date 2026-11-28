# Space-time representations


## Overview
Space-time representations model dynamic scenes by extending the query space of the rendering function to include time, omitting the explicit deformation network used in warp-based methods [S1]. Unlike warp-based approaches that separate geometry into a static canonical form and a dynamic transformation, space-time methods use a single unified function to directly regress dynamic properties by conditioning a 3D representation with a time component [S1]. This approach is tolerant of topology changes and can model a wider range of motion but is prone to time inconsistency [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A query tuple $q = (p, \mathbf{d}, t)$, where $p \in \mathbb{R}^3$ is the spatial position, $\mathbf{d}$ is the viewing direction, and $t$ is the time coordinate [S1].
*   **Output:** A property vector $\xi$ (e.g., color, density, or feature vector) [S1].

**Data Structures:**
The representation is implemented as either a neural implicit model conditioned on a time variable, a set of 3D Gaussians with time-dependent parameters, or an explicit feature grid [S1].

**Algorithm:**
1.  **Define the Query Function:** The method uses a single unified function to directly regress dynamic properties [S1]. Two common formulations exist [S1]:
    
    *Formulation A (Time in Main Function):*
    $$f_{\theta} : (\gamma(p, \mathbf{d}, t)) \rightarrow \xi \quad (12)$$
    Here, $\gamma$ is an encoding function that processes position, direction, and time jointly, and $\xi$ is the output property [S1].

    *Formulation B (Time in Encoding):*
    $$f : (\text{enc}(p, t), [\text{enc}(\mathbf{d})]) \rightarrow \xi \quad (13)$$
    Here, the encoding function $\text{enc}$ depends on time, while the main network $f$ processes the encoded spatial and directional features [S1].

2.  **Discretize Time:** Divide the time interval into equally spaced time slots [S1].

3.  **Compute Features at Time Slots:** Compute features at these discrete time slots using an MLP-based neural network or an explicit feature grid [S1].

4.  **Interpolate for Arbitrary Time:** Compute the feature of a point $p \in \mathbb{R}^3$ at any time $t$ by interpolating between the features of the neighboring time slots [S1].

5.  **Enforce Time Consistency (Optional):** Enforce time consistency locally by using flow fields, which define forward and backward mappings between neighboring time steps when available [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_space_time_representation(p, d, t, time_slots, enc_net, main_net):
    """
    Query a space-time representation for a property at position p, direction d, and time t.
    Implements Formulation B: time-dependent encoding with interpolation between slots.
    """
    # 1. Find neighboring time slots
    k = 0
    while k < len(time_slots) - 1 and time_slots[k + 1] <= t:
        k += 1
    t_k = time_slots[k]
    t_k1 = time_slots[min(k + 1, len(time_slots) - 1)]
    
    # 2. Compute features at neighboring slots using an MLP or feature grid [S1]
    feat_k = enc_net(concatenate(p, t_k))
    feat_k1 = enc_net(concatenate(p, t_k1))
    
    # 3. Interpolate features for arbitrary time t
    alpha = (t - t_k) / (t_k1 - t_k) if t_k1 != t_k else 0.0
    interpolated_feat = feat_k + alpha * (feat_k1 - feat_k)
    
    # 4. Encode direction (static)
    enc_d = enc_net(d)
    
    # 5. Main network regression
    xi = main_net(concatenate(interpolated_feat, enc_d))
    return xi
# Optional: Enforce time consistency locally using flow fields [S1]
```

**Parameters:**
*   **Time slots:** Equally spaced intervals used to discretize the time domain for feature computation [S1].

**Edge Cases and Pitfalls:**
*   **Time Inconsistency:** Due to the flexibility of the representation, the model may produce inconsistent results across time, especially if the number of observations is limited [S1].

## Subtopics

### [Warp-based Dynamic Scene Representation](6886-warp-based-dynamic-scene-representation.md)
Models temporal changes by learning a temporally global 3D canonical representation and a learnable, time-dependent deformation field. Sets itself apart by explicitly separating geometry and appearance, ensuring long-term correspondence. Choose when topology is stable and long-term consistency is critical.

### [Fixed Basis for Time Axis](10065-fixed-basis-for-time-axis.md)
A failed design that models temporal changes by storing a static feature matrix for each spatial point and computing features via an inner product with time-dependent basis functions. It differs by attempting to represent the entire volume as a linear combination of static basis volumes, which proved intractable due to memory and artifacts.

## Operating characteristics
*   **Time Complexity:** Using grid interpolation between time slots decreases the computational burden and enables faster training compared to evaluating an MLP at every time step [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Tolerant of topology changes; can model a wider range of motion than warp-based methods; omits the explicit deformation network [S1].
*   **Limitations:** Prone to time inconsistency; performance depends on the number of observations available [S1].
*   **When to Use:** When the scene undergoes topology changes or when a simpler pipeline without explicit deformation fields is preferred [S1].
*   **When Not to Use:** When long-term temporal consistency is critical, as the representation is prone to time inconsistency [S1].

## Background
Space-time representations emerged as an alternative to warp-based dynamic scene representations, aiming to simplify the modeling of dynamic scenes by directly incorporating time into the query space [S1]. The approach builds on the idea of conditioning 3D representations with time, allowing for a unified function to regress dynamic properties [S1]. Recent advances have explored both neural implicit models and explicit feature grids to implement this paradigm [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
