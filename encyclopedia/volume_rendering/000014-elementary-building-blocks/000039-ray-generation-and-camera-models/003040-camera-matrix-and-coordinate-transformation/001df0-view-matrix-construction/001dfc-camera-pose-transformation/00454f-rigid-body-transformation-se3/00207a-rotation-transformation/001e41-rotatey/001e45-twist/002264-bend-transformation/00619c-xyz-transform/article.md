# XYZ Transform

Also called: XYZ Transform module, Coordinate Transformation

## Overview
The XYZ Transform is a module that modifies 3D coordinate space before it reaches a 3D Mask, enabling spatial operations such as moving, rotating, repeating, folding, wrapping, bending, twisting, or distorting (via noise or sine functions) the input coordinates [S1]. It sets itself apart from sibling methods by acting as a general-purpose coordinate modifier that feeds into specific shape definitions (Masks), allowing a single shape definition to produce complex, repeated, or distorted instances without altering the underlying distance field logic [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D position vector $\mathbf{p} = (x, y, z)$ representing the current coordinate in space.
*   **Output:** A transformed 3D position vector $\mathbf{p}' = (x', y', z')$ passed to the downstream 3D Mask.

**Data Structures:**
The module operates on standard 3D floating-point vectors. No specific bit-packing or complex data structures are defined in the sources; it functions as a direct coordinate mapping function [S1].

**Steps:**
1.  The XYZ Transform module receives the current XYZ position $\mathbf{p}$ from the raymarching or evaluation pipeline.
2.  The module applies a transformation function $T$ to $\mathbf{p}$, producing $\mathbf{p}' = T(\mathbf{p})$. Common transformations include:
    *   **Repeat:** Modulo arithmetic to tile space.
    *   **Rotate:** Linear rotation matrices.
    *   **Distort:** Non-linear functions such as sine waves or noise fields.
3.  The transformed coordinates $\mathbf{p}'$ are passed to the 3D Mask.
4.  The 3D Mask calculates the distance to its shape using $\mathbf{p}'$ instead of $\mathbf{p}$. The resulting distance field represents the shape in the transformed space [S1].

**Formulas:**
While specific formulas for all variants are not provided, the general operation is:
$$ \mathbf{p}' = T(x, y, z) $$
Where $T$ is the specific transformation function applied by the module [S1].

**Parameters:**
Parameters depend on the specific transformation type (e.g., rotation angle, repeat frequency, noise amplitude). Typical values are not specified in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Continuity:** Discontinuous transformations (like hard wrapping) may introduce artifacts in the distance field gradient if not handled carefully by the downstream renderer [S1].
*   **Complexity:** Highly distorted coordinates may require more sampling steps to resolve the resulting complex geometry accurately [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def xyz_transform(position, transform_type, params):
    """
    Applies a coordinate transformation to a 3D position.
    
    Args:
        position: tuple (x, y, z)
        transform_type: string indicating the operation (e.g., 'repeat', 'rotate')
        params: dictionary of parameters for the transformation
        
    Returns:
        tuple (x', y', z') transformed position
    """
    x, y, z = position
    
    if transform_type == 'repeat':
        # Example: Repeat coordinates based on a frequency factor
        freq = params.get('frequency', 1.0)
        x_prime = x % freq
        y_prime = y % freq
        z_prime = z % freq
        return (x_prime, y_prime, z_prime)
        
    elif transform_type == 'rotate':
        # Example: Rotate around Y-axis by angle theta
        theta = params.get('angle', 0.0)
        cos_t = cos(theta)
        sin_t = sin(theta)
        x_prime = x * cos_t - z * sin_t
        y_prime = y
        z_prime = x * sin_t + z * cos_t
        return (x_prime, y_prime, z_prime)
        
    elif transform_type == 'distort_sine':
        # Example: Distort using sine wave
        amplitude = params.get('amplitude', 0.1)
        frequency = params.get('frequency', 1.0)
        x_prime = x + amplitude * sin(frequency * y)
        y_prime = y
        z_prime = z
        return (x_prime, y_prime, z_prime)
        
    else:
        # Default: Identity transformation if type is unknown
        return (x, y, z)
```

## Subtopics

### [Multi-scale Coordinate Transformation](19793-multi-scale-coordinate-transformation.md)
Multi-scale Coordinate Transformation is a method for enhancing the representational capacity of neural fields by applying a single coordinate transformation function $\gamma_i$ at multiple spatial resolutions. It solves the problem of modeling signals that contain both low-frequency smooth components and high-frequency fine details by decomposing the signal into a product of spatially varying coefficient maps and multi-level basis maps. It sets itself apart from single-scale transformations by explicitly encoding frequency information through scaled inputs, allowing the network to capture a broader spectrum of signal characteristics.

## Operating characteristics
*   **Time Complexity:** Depends on the complexity of the transformation function $T$. Simple linear transformations (rotation, translation) are $O(1)$. Non-linear transformations (noise, sine) are also $O(1)$ but with higher constant factors.
*   **Memory Complexity:** $O(1)$ as it operates on a per-vertex or per-ray basis without storing large state structures.
*   **Strengths:** Allows for complex spatial arrangements (repetition, distortion) without modifying the underlying shape definitions. Enables modular composition of effects [S1].
*   **Limitations:** The resulting distance field may have complex gradients, potentially requiring more sophisticated rendering techniques (e.g., adaptive sampling) for accurate visualization [S1].
*   **When to use:** When you need to tile, rotate, or distort a base shape without creating multiple separate shape definitions.
*   **When not to use:** When the transformation introduces discontinuities that cannot be resolved by the downstream distance field renderer.

## Background
The XYZ Transform is part of the "3D Shaders" module set within Drambo Visual, a software environment for creating 3D visual effects and shaders [S1]. It is designed to work in conjunction with 3D Masks and Mask Processor modules to build complex distance fields [S1]. The concept is analogous to 2D coordinate transformations used before 2D masks, but extended to 3D space [S1].

## Sources
[S1] 3D Shaders - Drambo Visual, https://www.beepstreet.com/drambo-visual/3d-shaders.html, fetched 2026-10-30
