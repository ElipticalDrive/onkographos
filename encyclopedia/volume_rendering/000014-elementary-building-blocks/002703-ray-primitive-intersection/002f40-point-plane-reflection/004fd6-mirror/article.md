# Mirror

Also called: Mirror Tool, Mirroring tool

## Overview
The Mirror method is a volumetric filling tool that copies content from a source region to a target region across a user-defined reflection plane [S1]. It is designed to exploit symmetry in structures, such as the human body or head, by allowing users to interactively define a plane that divides the volume into two parts [S1]. Unlike the parent [Point-Plane Reflection](5920-point-plane-reflection.md) which reflects a single point, Mirror operates on volumetric data, copying material values from one side of the plane to the other [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a user-defined reflection plane (position and orientation), and a mirroring direction [S1].
*   **Output:** A modified volume where the target region contains copied material from the source region [S1].

**Data Structures:**
The implementation uses a texture-based system. The volume data is stored in `VolumeTex` (3D texture) [S1]. A `SculptMap` (3D texture) acts as a mask representing visibility and inserted material, using the alpha channel (`Col.a`) for visibility and the blue channel (`Col.b`) for inserted material [S1]. A `PolygonCoord` (2D texture) maps fragment coordinates to the volume [S1].

**Steps:**
1.  The user interactively selects the position and orientation of the reflection plane [S1].
2.  The direction of mirroring, represented by an arrow on the plane, defines the target area for copying [S1]. The opposite direction defines the source region [S1].
3.  For each voxel $V$ in the selected target part of the volume, calculate the coordinates of its mirrored voxel $V'$ [S1].
    *   Let $P$ be the orthogonal projection of $V$ onto the reflection plane [S1].
    *   Let $d$ be the distance between voxel $V$ and its orthogonal projection $P$ [S1].
    *   The mirrored voxel $V'$ is located at an absolute distance $|d|$ from $P$, in the same direction as the vector $\vec{v}(V-p)$ [S1]. Note: The source text uses $p$ in the vector definition, likely referring to a point on the plane or the projection point $P$ itself, but explicitly states "vector v(V-p)" [S1].
4.  Access the voxel at $V'$ in the source region [S1].
5.  If visible material exists at $V'$, copy its content to $V$ [S1].

**Formula:**
Let $d$ be the distance between voxel $V$ and its orthogonal projection $P$ on the reflection plane. The mirrored voxel $V'$ is located at absolute distance $|d|$ from $P$, in the same direction as the vector $\vec{v}(V-p)$ [S1].

**Pseudocode:**
The following pseudocode represents the fragment shader `FSMirror` described in the source, which processes fragments of a proxy geometry to update the volume mask and material [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs
# P: Uniform float4 representing the 3D point 'P' (plane parameters)
# SculptMap: 3D texture for visibility and insertion masks
# PolygonCoord: 2D texture for texture coordinates
# VolumeTex: 3D texture for the volume data

# Output
# Col: float4 color output

def FSMirror(v2f_TexCoord, P, SculptMap, PolygonCoord, VolumeTex):
    # Map fragment to volume coordinates
    texCoord = tex2D(PolygonCoord, v2f_TexCoord)
    
    # Get current volume voxel value
    volume_voxel = tex3D(VolumeTex, texCoord.xyz)
    
    # Get current mask state (visibility in alpha, insertion in blue)
    Col = tex3D(SculptMap, texCoord.xyz)
    
    # Calculate 3D Euclidean distance from the plane point P to the current voxel
    # P.a is used as a divisor, likely related to plane normalization or scaling
    D = distance(float4(P.xyz, 1.0), texCoord) / P.a
    
    # Check if the voxel is within the influence of the mirror tool
    if D < Col.a:
        # Do not overwrite existing material if it is not background
        # Col.a == 0.0f implies no visibility/mask active
        # volume_voxel.r == 0.0f implies background/no material
        if volume_voxel.r == 0.0f or Col.a == 0.0f:
            # Mark this voxel as having inserted material (using blue channel)
            Col.b = 1.0f
            
    return Col
```

**Edge Cases and Pitfalls:**
*   **Overwriting:** The tool does not overwrite existing material. It checks the current value of voxels; they are filled only if they have background values (e.g., `volume_voxel.r == 0.0f`) [S1].
*   **Visibility:** The tool relies on a visibility mask (`Col.a`) to determine which voxels are affected [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Exploits symmetry in structures like the human body, reducing manual effort for symmetric modeling [S1]. Integrated into a texture-based volume rendering system, allowing users to manipulate only a 3D mask before concluding the sculpting process [S1].
*   **Limitations:** Limited to structures with symmetry [S1]. Does not overwrite existing material, which may require pre-cleaning of the target region [S1].

**When to Use:**
Use when working with symmetric anatomical structures (e.g., human head) and when the target region is empty or contains background values [S1].

## Background
The Mirror tool was developed as part of a volume sculpting system based on geometric tools, specifically integrated with 3D Clipper and Digger tools [S1]. It was designed to provide direct insertion of material in voxels by exploiting the existing symmetry of many structures of the human body [S1]. The implementation uses specific shaders (`FSMirror`) applied to fragments of proxy geometry within a texture-based volume rendering pipeline [S1]. The tool was assessed in a practical application reported in Section 7 of the source paper [S1].

## Sources
[S1] Volume sculpting based on geometric tools, https://doi.org/10.1007/bf03194498, fetched 2026-10-26
