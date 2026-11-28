# Mesh to DRK Conversion

Also called: Mesh2DRK

## Overview
Mesh to DRK Conversion is a training-free procedure that converts traditional triangle and polygon mesh faces into Deformable Radial Kernel (DRK) primitives for seamless integration into radiance field scenes [S1]. Unlike existing splatting methods (2D-GS, 3D-GS, 3D-HGS, GES) which fail to integrate traditional assets seamlessly, this method provides a direct, training-free conversion pipeline [S1]. It bridges the gap between millions of traditional 3D assets and fast, high-quality reconstruction representations, enabling unified rendering, VR digital world creation, and AR scene enhancement [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A traditional 3D mesh $\mathcal{M}$ defined by polygon faces (typically triangles) [S1].
*   **Output:** A list of DRK primitives representing the mesh geometry [S1].

**Data Structures:**
Each primitive is defined by its endpoints, sharpness parameters, and curvature parameters [S1]. The specific internal layout of a DRK primitive is not detailed in the sources, but the conversion logic maps mesh vertices directly to DRK endpoints [S1].

**Steps:**
1.  **Vertex Mapping:** For each polygon face in the input mesh, place the DRK endpoints directly at the polygon's vertices [S1].
2.  **Curvature Initialization:** Set both the L1 and L2 curvature parameters for all generated DRK primitives to 1 [S1].
3.  **Sharpness Tuning:** Adjust the parameters $\tau$ and $\eta$ to control the sharpness of the resulting DRK primitives [S1].

**Parameters:**
*   $\tau$: Sharpness control parameter for DRK primitives.
*   $\eta$: Sharpness control parameter for DRK primitives.
*   L1 Curvature: Fixed value of 1 [S1].
*   L2 Curvature: Fixed value of 1 [S1].
Both $\tau$ and $\eta$ are tuned per asset; no fixed default is given [S1].

**Pitfalls:**
The conversion process relies on fixed curvature values (L1=1, L2=1) that may not adapt to complex local geometry without manual tuning of $\tau$ and $\eta$ [S1]. While this avoids convergence issues associated with training-based methods, it requires manual adjustment for optimal results on complex assets [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_to_drk(mesh):
    """
    Converts a mesh into DRK primitives.
    
    Args:
        mesh: A mesh object with attributes 'faces' (list of dicts with 'vertex_indices')
              and 'vertices' (list of 3D position arrays).
    
    Returns:
        drk_primitives: A list of DRK primitive dictionaries.
    """
    drk_primitives = []
    
    for face in mesh.faces:
        vertex_indices = face['vertex_indices']
        vertices = [mesh.vertices[i] for i in vertex_indices]
        primitive = {
            'endpoints': vertices,
            'tau': 1.0,
            'eta': 1.0,
            'l1_curvature': 1.0,
            'l2_curvature': 1.0
        }
        drk_primitives.append(primitive)
        
    return drk_primitives
```

## Operating characteristics
*   **Time Complexity:** The conversion is $O(N)$ where $N$ is the number of faces in the mesh, as it involves a direct mapping of vertices to primitives without iterative optimization [S1].
*   **Memory Complexity:** The memory usage is proportional to the number of primitives generated, which is directly related to the mesh resolution (number of faces/vertices) [G].
*   **Strengths:** The method is training-free, avoiding convergence issues and the need for training data rendering or model optimization [S1]. It allows for the effortless conversion of traditional assets into DRK scenes [S1].
*   **Limitations:** It relies on fixed curvature values (L1=1, L2=1) which may not adapt to complex local geometry without manual tuning of $\tau$ and $\eta$ [S1].
*   **When to use:** When integrating traditional 3D assets (e.g., flowers, leaves, detailed textures) into DRK scenes for unified rendering, VR, or AR applications [S1].
*   **When not to use:** When high-fidelity adaptation to complex local geometry is required without manual parameter tuning, or when the asset is already in a format compatible with other splatting methods that might offer better automatic adaptation [G].

## Background
The method was introduced in the paper "Deformable Radial Kernel Splatting" [S1]. It addresses the limitation of existing splatting methods (2D-GS, 3D-GS, 3D-HGS, GES) in seamlessly integrating with traditional 3D assets [S1]. The DRK framework provides a versatile representation that allows for this conversion, bridging the gap between traditional 3D assets and fast, high-quality reconstruction representations [S1].

## Sources
[S1] Deformable Radial Kernel Splatting, https://doi.org/10.1109/CVPR52734.2025.02004, fetched 2026-10-30
