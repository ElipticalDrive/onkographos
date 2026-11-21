# Vector Displacement

Also called: Vector Texture Displacement

## Overview
Vector Displacement is a procedural technique in volume rendering that shifts fluid or surface points based on a colored vector texture map, where the texture's RGB channels are interpreted as a 3D displacement vector [S1]. It solves the limitation of scalar displacement methods by enabling the creation of asymmetric, complex surface geometries, such as waves with convex backs and concave fronts, which scalar methods cannot produce [S1]. Unlike Surface-driven displacement, which relies on gradient fields and produces symmetrical forms, Vector Displacement allows for directional control of the displacement magnitude and orientation directly through the texture data [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A colored vector texture map (RGB) containing values that can be negative or positive. The interpretation of these values depends on the rendering mode (Mesh vs. Non-Mesh) [S1].
*   **Output:** A displaced volume or surface geometry where each point is shifted by the vector defined by the texture sample at that location [S1].

**Data Structures:**
The displacement map is a 2D or 3D texture where each texel stores three floating-point or normalized integer values corresponding to the X, Y, and Z components of the displacement vector [S1].

**Parameters:**
*   `displ2d`: Specifies the displacement technique type. Set to "Vector" [S1].
*   `displmul`: Specifies the displacement multiplier, scaling the magnitude of the displacement vector [S1].
*   `displ2`: Specifies the displacement map texture [S1].

**Steps:**
1.  **Sample Texture:** For each point on the fluid volume or surface, sample the displacement map at the corresponding spatial coordinates to retrieve a color value $C = (R, G, B)$ [S1].
2.  **Interpret Vector:**
    *   If the rendering **Mode** is *Mesh*, *Ocean Mesh*, or *Cap Mesh*: The texture must be in V-Ray Tangent Vector format. The components are interpreted as:
        *   $X = R - 0.5$
        *   $Y = G - 0.5$
        *   $Z = B - 0.0$
        *   Here, gray ($R=0.5, G=0.5$) and black ($B=0.0$) result in zero displacement. Brighter values move points along positive axes; darker/negative values move points along negative axes [S1].
    *   If the rendering **Mode** is *other than mesh modes*: The texture is 0.0-based. The components are interpreted directly as:
        *   $X = R$
        *   $Y = G$
        *   $Z = B$
        *   Black ($0,0,0$) results in no displacement. Brighter colors shift points towards positive axes; negative colors shift along negative axes [S1].
3.  **Scale Displacement:** Multiply the resulting vector $\vec{V} = (X, Y, Z)$ by the multiplier parameter `displmul` [S1].
4.  **Apply Shift:** Shift the point's position by the scaled vector $\vec{V}_{scaled}$ [S1].

**Pitfalls and Edge Cases:**
*   **Mode Mismatch:** If the Mode is set to Mesh, Ocean Mesh, or Cap Mesh, the texture *must* be in the V-Ray Tangent Vector format (X/Y 0.5-based, Z 0.0-based). Using a standard 0.0-based texture in these modes will result in incorrect or unintended displacement [S1].
*   **Texture Format:** If the Mode is not a mesh mode, the texture *must* be 0.0-based. Using a V-Ray Tangent Vector format texture in non-mesh modes will cause incorrect displacement behavior [S1].
*   **Monochrome Input:** If a monochrome map is passed when a vector map is required (e.g., for Advection displacement), the displacement effect will point in a single direction, as the color channels are not independent [S1].

## Subtopics

### Advection Displacement
Advection Displacement is a variant of Vector Displacement that also requires a colored 0.0-based vector texture map with negative and positive values [S1]. It is functionally similar to Vector Displacement but is specifically optimized to avoid producing grainy structures in fire and smoke simulations [S1]. It is commonly combined with a Phoenix Grid Texture (with Channel set to Velocity) to produce render-time gridless advection effects [S1]. A key pitfall is that passing a monochrome map instead of a vector map causes the displacement to point in a single uniform direction [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Vector Displacement is generally faster than Gradient driven displacement, which is noted as being slower [S1].
*   **Strengths:** Capable of producing more complicated, asymmetric surfaces (e.g., convex back/concave front waves) compared to the symmetrical forms produced by Surface-driven displacement [S1].
*   **Limitations:** Requires specific texture formats depending on the rendering mode (Mesh vs. Non-Mesh). Incorrect texture format leads to erroneous displacement. Monochrome maps cannot drive directional displacement effectively [S1].
*   **Use Cases:** Ideal for simulating ocean waves, fluid surfaces with directional flow, and fire/smoke advection where grainy artifacts from scalar methods are undesirable [S1].

## Background
Vector Displacement is a feature within the Phoenix FD plugin for 3ds Max, designed to enhance the visual fidelity of fluid simulations by allowing procedural displacement during rendering [S1]. It leverages vector fields (such as velocity) to drive geometric changes, moving beyond simple scalar-based surface offsets [S1].

## Sources
[S1] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
