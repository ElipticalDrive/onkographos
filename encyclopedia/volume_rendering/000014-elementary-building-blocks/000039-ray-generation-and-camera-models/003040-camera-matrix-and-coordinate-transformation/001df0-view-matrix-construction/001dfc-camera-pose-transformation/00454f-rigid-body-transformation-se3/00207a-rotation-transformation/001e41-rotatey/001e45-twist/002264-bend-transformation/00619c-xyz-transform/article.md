# XYZ Transform


## Overview
The XYZ Transform is a module that modifies 3D coordinate space before it reaches a 3D Mask, enabling spatial operations such as moving, rotating, repeating, folding, wrapping, bending, twisting, or distorting (via noise or sine functions) the input coordinates [S1]. It sets itself apart from sibling methods by acting as a general-purpose coordinate modifier that feeds into specific shape definitions (Masks), allowing a single shape definition to produce complex, repeated, or distorted instances without altering the underlying distance field logic [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D position vector $\mathbf{p} = (x, y, z)$ representing the current coordinate in space [S1].
*   **Output:** A transformed 3D position vector $\mathbf{p}' = (x', y', z')$ passed to the downstream 3D Mask [S1].

**Data Structures:**
The module operates on 3D coordinate vectors [S1]. No specific bit-packing or complex data structures are defined in the sources [S1].

**Steps:**
1. The XYZ Transform module receives the current XYZ position $\mathbf{p}$ from the raymarching or evaluation pipeline [S1].
2. The module modifies the coordinates (e.g., repeating them) [S1].
3. The 3D Mask receives the transformed coordinates and calculates the same shape in its own space, but the result becomes modified (e.g., many spheres instead of one) [S1].

**Formulas:**
While specific formulas for all variants are not provided, the general operation is:
$$ \mathbf{p}' = T(x, y, z) $$
Where $T$ is the specific transformation function applied by the module [S1].

**Parameters:**
Parameters depend on the specific transformation type (e.g., rotation angle, repeat frequency, noise amplitude) [S1]. Typical values are not specified in the sources [S1].

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

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
        cos_t = math.cos(theta)
        sin_t = math.sin(theta)
        x_prime = x * cos_t - z * sin_t
        y_prime = y
        z_prime = x * sin_t + z * cos_t
        return (x_prime, y_prime, z_prime)
        
    elif transform_type == 'distort_sine':
        # Example: Distort using sine wave
        amplitude = params.get('amplitude', 0.1)
        frequency = params.get('frequency', 1.0)
        x_prime = x + amplitude * math.sin(frequency * y)
        y_prime = y
        z_prime = z
        return (x_prime, y_prime, z_prime)
        
    else:
        # Default: Identity transformation if type is unknown
        # Note: Exact formulas for move, fold, wrap, bend, twist, and noise distortions are not provided in the sources.
        return (x, y, z)
```

## Subtopics

### [Multi-scale Coordinate Transformation](19793-multi-scale-coordinate-transformation.md)
Multi-scale Coordinate Transformation enhances neural field representational capacity by applying a single coordinate transformation function $\gamma_i$ at multiple spatial resolutions. It solves the problem of modeling signals containing both low-frequency smooth components and high-frequency fine details by decomposing the signal into spatially varying coefficient maps and multi-level basis maps. It sets itself apart from single-scale transformations by explicitly encoding frequency information through scaled inputs, allowing the network to capture a broader spectrum of signal characteristics.

## Operating characteristics
Not given in the sources.

## Background
The method is documented in the 3D Shaders section for Drambo Visual [S1]. It is designed to work in conjunction with 3D Masks and Mask Processor modules to build complex distance fields [S1]. The concept is analogous to 2D coordinate transformations used before 2D masks, but extended to 3D space [S1].

## Sources
[S1] 3D Shaders - Drambo Visual, https://www.beepstreet.com/drambo-visual/3d-shaders.html, fetched 2026-10-30
