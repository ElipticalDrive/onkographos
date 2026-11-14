# Zooming Interaction Model

Also called: Shift of center of the object, Shift of the center of the object, Zooming Interaction Transform

## Overview
The Zooming Interaction Model is a camera transformation technique used in volume visualization to shift the center of the rendered object during zooming operations [S1, S2, unofficial]. It solves the problem of maintaining user attention on specific focus parts of the volume by dynamically adjusting the model-view matrix based on the initial interaction point [S1, S2, unofficial]. Unlike standard zooming which typically scales uniformly around the view center or object origin, this model emphasizes the region near the initial button click by translating the view [S1, S2, unofficial]. It is a component of the Volume Visualization Camera, which handles scaling, centering, rotation, and translation [Parent].

## How it works
The method modifies the current model-view matrix ($MV$) to produce a new model-view matrix ($MV'$) that shifts the view origin. This ensures that the area of interest, defined by the user's initial interaction, remains the focal point as the magnification changes [S1, S2, unofficial].

**Inputs and Outputs:**
- Input: Original model-view matrix $MV$, magnification factor $R$, interaction point $\vec{P}_{interaction}$, rendering area center $\vec{P}_{center}$.
- Output: Modified model-view matrix $MV'$.

**Data Structures:**
- $MV$: A $4 \times 4$ transformation matrix representing the current view and model orientation.
- $MV'$: A $4 \times 4$ transformation matrix representing the updated view.
- $\vec{P}_{interaction}$: A 2D or 3D vector representing the coordinates of the initial user interaction (e.g., mouse click position).
- $\vec{P}_{center}$: A 2D or 3D vector representing the center of the current rendering area.
- $R$: A scalar float representing the magnification factor.

**Formulas:**
The transformation is defined by Equation 1 [S1, S2, unofficial]:

$$MV' = \text{Translate}(-R \cdot (\vec{P}_{interaction} - \vec{P}_{center})) \cdot MV$$

Where:
- $\text{Translate}(\vec{v})$ is a translation matrix that shifts an object by vector $\vec{v}$ [S1, S2, unofficial].
- $\vec{P}_{interaction}$ is the position of the initial user interaction [S1, S2, unofficial].
- $\vec{P}_{center}$ is the center of the rendering area [S1, S2, unofficial].
- $R$ is the magnification factor [S1, S2, unofficial].

**Parameters:**
- $R$: Magnification factor. Typical values depend on the zoom level; $R > 1$ for zooming in, $R < 1$ for zooming out. Not given in the sources for specific standard values.
- $\vec{P}_{interaction}$: Position of the initial user interaction.
- $\vec{P}_{center}$: Center of the rendering area.

**Steps:**
1. Calculate the displacement vector $\vec{d} = \vec{P}_{interaction} - \vec{P}_{center}$.
2. Scale the displacement vector by the magnification factor: $\vec{v} = -R \cdot \vec{d}$.
3. Construct a translation matrix $T = \text{Translate}(\vec{v})$.
4. Multiply the translation matrix by the original model-view matrix: $MV' = T \cdot MV$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def zooming_interaction_model(MV, R, P_interaction, P_center):
    """
    Computes the modified model-view matrix for the Zooming Interaction Model.
    
    Args:
        MV (Matrix4x4): The original model-view matrix.
        R (float): The magnification factor.
        P_interaction (Vector3): The position of the initial user interaction.
        P_center (Vector3): The center of the rendering area.
        
    Returns:
        Matrix4x4: The modified model-view matrix MV'.
    """
    # Step 1: Calculate displacement vector
    # Note: Vector subtraction assumes compatible dimensions (e.g., 3D or 2D homogeneous)
    displacement = P_interaction - P_center
    
    # Step 2: Scale by magnification factor and negate
    # The negative sign shifts the view to keep the interaction point in focus
    translation_vector = -R * displacement
    
    # Step 3: Construct translation matrix
    # A standard 4x4 translation matrix
    T = Matrix4x4(
        1, 0, 0, translation_vector.x,
        0, 1, 0, translation_vector.y,
        0, 0, 1, translation_vector.z,
        0, 0, 0, 1
    )
    
    # Step 4: Multiply translation matrix by original model-view matrix
    MV_prime = T @ MV
    
    return MV_prime
```

**Edge Cases and Pitfalls:**
- Dimensionality: $\vec{P}_{interaction}$ and $\vec{P}_{center}$ must be in the same coordinate space as the translation component of $MV$. If they are 2D screen coordinates, they must be projected into the view space or handled appropriately before matrix multiplication [S1, S2, unofficial].
- Precision: Large magnification factors $R$ may lead to floating-point precision issues in the translation vector, potentially causing jitter or artifacts in the rendered volume.
- Initial Interaction: The model relies on the "initial button click" position. If the interaction point is not tracked correctly across zoom events, the shift will be incorrect [S1, S2, unofficial].

## Subtopics

### Volume Visualization Camera
A camera controller for volume visualization that computes a view matrix by applying volume scaling, centering, rotation, zoom, and translation transformations. It provides the framework within which the Zooming Interaction Model operates. [Parent]

## Operating characteristics
- **Time Complexity**: $O(1)$ per zoom event, as it involves a fixed number of vector operations and a single matrix multiplication.
- **Memory Complexity**: $O(1)$, requiring storage for the displacement vector and the translation matrix.
- **Strengths**: Effectively emphasizes user attention on specific focus parts of the object near the initial button click [S1, S2, unofficial]. Simple to implement.
- **Limitations**: Only applies to zooming interactions; panning and rotation do not use this semantic-dependent navigation [S1, S2, unofficial]. The shift is based on the initial click, which may not be optimal for continuous zooming if the user's focus changes.
- **When to use**: When the user's attention is expected to remain on the region near the initial zoom interaction point.
- **When not to use**: When the user needs to pan or rotate without shifting the center, or when the interaction point is not well-defined.

## Background
The Zooming Interaction Model was introduced in the context of Dynamic Focus + Context for Volume Rendering [S1, S2, unofficial]. It is part of a system that includes engagement estimation modules for rotation, panning, and zooming interactions [S1, S2, unofficial]. The rotation and panning interactions in this system are based on a trackball metaphor [S1, S2, unofficial]. The model assumes that during a zooming operation, the user is most interested in the parts of the object near the initial button click [S1, S2, unofficial]. This allows the system to emphasize the user's attention on the focus parts of the object [S1, S2, unofficial].

## Sources
[S1] Dynamic Focus + Context for Volume Rendering, https://doi.org/10.2312/pe/vmv/vmv10/331-338, fetched 2026-10-26 (unofficial)
[S2] (PDF) Dynamic Focus+Context for Volume Rendering - ResearchGate, https://www.researchgate.net/publication/220839241_Dynamic_FocusContext_for_Volume_Rendering, fetched 2026-10-27 (unofficial)
