# Mesh to DRK Conversion

Also called: Mesh2DRK

## Overview
Mesh to DRK Conversion is a training-free procedure that converts traditional triangle and polygon mesh faces into Deformable Radial Kernel (DRK) primitives for seamless integration into radiance field scenes [S1]. Unlike existing splatting methods (2D-GS, 3D-GS, 3D-HGS, GES) which fail to integrate traditional assets seamlessly, this method provides a direct, training-free conversion pipeline [S1]. It bridges the gap between millions of traditional 3D assets and fast, high-quality reconstruction representations, enabling unified rendering, VR digital world creation, and AR scene enhancement [S1].

## How it works
**Inputs:** A traditional 3D mesh consisting of triangle and polygon faces.
**Outputs:** A set of DRK primitives corresponding to the input mesh geometry.

**Data Structures:**
The input mesh is defined by a set of vertices $V$ and faces $F$. Each face $f \in F$ is a polygon defined by its vertices. The output is a collection of DRK primitives, where each primitive is associated with a face or vertex of the mesh.

**Parameters:**
*   $\tau$: Sharpness control parameter for DRK primitives. Typical value: Tuned per asset; no fixed default given [S1].
*   $\eta$: Sharpness control parameter for DRK primitives. Typical value: Tuned per asset; no fixed default given [S1].
*   $L1$: L1 curvature parameter for DRK. Fixed value: 1 [S1].
*   $L2$: L2 curvature parameter for DRK. Fixed value: 1 [S1].

**Steps:**
1.  Iterate through each polygon face in the input mesh.
2.  For each polygon face, place the DRK endpoints directly at the polygon's vertices [S1].
3.  Set the L1 curvature parameter to 1 [S1].
4.  Set the L2 curvature parameter to 1 [S1].
5.  Adjust the parameters $\tau$ and $\eta$ to control the sharpness of the resulting DRK primitives [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_to_drk_conversion(mesh):
    """
    Converts a mesh to DRK primitives.
    
    Args:
        mesh: An object containing 'vertices' (list of 3D points) 
              and 'faces' (list of lists of vertex indices).
              
    Returns:
        drk_primitives: A list of DRK primitive objects.
    """
    drk_primitives = []
    
    # Fixed curvature parameters as per source [S1]
    L1_curvature = 1
    L2_curvature = 1
    
    # Sharpness parameters need to be tuned per asset [S1]
    # These are placeholders for user-defined values
    tau = 1.0 
    eta = 1.0
    
    for face in mesh.faces:
        # Get vertices of the current polygon face
        face_vertices = [mesh.vertices[idx] for idx in face]
        
        # Create DRK primitive for this face
        # The DRK endpoints are placed directly at the polygon's vertices [S1]
        drk_primitive = DRKPrimitive(
            endpoints=face_vertices,
            tau=tau,
            eta=eta,
            L1=L1_curvature,
            L2=L2_curvature
        )
        
        drk_primitives.append(drk_primitive)
        
    return drk_primitives

class DRKPrimitive:
    def __init__(self, endpoints, tau, eta, L1, L2):
        self.endpoints = endpoints
        self.tau = tau
        self.eta = eta
        self.L1 = L1
        self.L2 = L2
```

**Edge Cases and Pitfalls:**
*   The conversion process requires no training data rendering or model optimization, avoiding convergence issues [S1].
*   However, it relies on fixed curvature values ($L1=1, L2=1$) that may not adapt to complex local geometry without manual tuning of $\tau$ and $\eta$ [S1].
*   If $\tau$ and $\eta$ are not tuned appropriately, the resulting primitives may not accurately represent the sharpness or detail of the original mesh geometry.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of faces in the mesh, as each face is processed independently to create a DRK primitive.
*   **Memory Complexity:** Linear with respect to the number of faces, storing one DRK primitive per face.
*   **Strengths:** Training-free, fast conversion, seamless integration of traditional 3D assets into DRK scenes [S1].
*   **Limitations:** Relies on fixed curvature parameters and manual tuning of sharpness parameters ($\tau, \eta$), which may not adapt automatically to complex local geometry [S1].
*   **When to use:** When integrating traditional 3D assets (e.g., flowers, leaves, detailed textures like tags and bark) into DRK scenes for unified rendering, VR, or AR applications [S1].
*   **When not to use:** When automatic adaptation to complex local geometry is required without manual parameter tuning.

## Background
The Mesh to DRK Conversion method was introduced in the paper "Deformable Radial Kernel Splatting" [S1]. It addresses the limitation of existing splatting methods (2D-GS, 3D-GS, 3D-HGS, GES) in seamlessly integrating with traditional assets [S1]. The DRK framework provides a versatile representation that allows for effortless conversion of triangle and polygon faces into DRK primitives [S1]. This eliminates the need for training data rendering or model optimization, enabling rich 3D assets to be seamlessly transferred into DRK [S1].

## Sources
[S1] Deformable Radial Kernel Splatting, https://doi.org/10.1109/CVPR52734.2025.02004, fetched 2026-10-30
