# Pressure Gradient Flow Control

Also called: External Pressure Controller, Pressure-based Fluid Flow

## Overview
Pressure Gradient Flow Control manipulates the direction of fluid surface flow by explicitly setting pressure values in Surface cells, causing fluid to move from high-pressure regions to low-pressure regions [S1, unofficial]. It differs from standard Navier-Stokes solvers by allowing direct artistic or physical control over surface topology without altering viscosity or density, and it is distinct from the Internal Pressure Controller which adds to existing pressures rather than defining the gradient direction for surface flow [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A grid/mesh where cells are classified as "Surface" or "Interior/Background". A scalar pressure field $P$ defined on these cells.
*   **Output:** Modified velocity field or force vector applied to Surface cells to drive flow according to the pressure gradient.

**Data Structures:**
*   **Grid:** A structured grid (e.g., voxel grid) where each cell has a type flag (`Surface`, `Solid`, `Fluid`).
*   **Pressure Field:** A scalar array $P[i,j,k]$ aligned with the grid cells.

**Steps:**
1.  **Identify Surface Cells:** Iterate through the grid to identify all cells marked as `Surface`.
2.  **Assign Pressure Values:** Set the pressure value $P$ for each Surface cell. Higher values indicate regions the fluid should flow away from; lower values indicate regions the fluid should flow toward [S1, unofficial].
3.  **Compute Gradient:** Calculate the pressure gradient $\nabla P$ at each Surface cell. For a cell at index $(i,j,k)$, the gradient is approximated using finite differences with neighbors:
    $$ \nabla P \approx \left( \frac{P_{i+1,j,k} - P_{i-1,j,k}}{2\Delta x}, \frac{P_{i,j+1,k} - P_{i,j-1,k}}{2\Delta y}, \frac{P_{i,j,k+1} - P_{i,j,k-1}}{2\Delta z} \right) $$
    Where $\Delta x, \Delta y, \Delta z$ are grid spacings.
4.  **Apply Force/Velocity:** Apply a force or velocity update proportional to the negative pressure gradient to drive flow from high to low pressure:
    $$ \mathbf{v}_{new} = \mathbf{v}_{old} - \alpha \nabla P $$
    Where $\alpha$ is a scaling factor controlling the strength of the effect [G].
5.  **Boundary Handling:** Ensure pressure values are consistent with solid boundaries handled by the parent solver. Pressure gradients should not drive fluid into solid cells [S1, unofficial].

**Parameters:**
*   **Pressure Magnitude:** Controls the intensity of the flow. Typical values depend on the scale of the simulation.
*   **Scaling Factor ($\alpha$):** Controls how strongly the pressure gradient influences velocity.

**Edge Cases and Pitfalls:**
*   **Discontinuities:** Sharp changes in pressure can cause numerical instability or unrealistic fluid behavior. Smooth pressure fields are preferred.
*   **Boundary Conditions:** Pressure values at solid boundaries must be handled carefully to prevent fluid from flowing into or out of solids incorrectly.
*   **Zero Gradient:** If pressure is uniform, no flow is induced.

## Subtopics

### Surface Tension Controller
Affects surface cohesion by creating a small negative pressure in each Surface cell, causing bonding between cells to form droplets or sheeting effects [S1, unofficial]. It differs from Pressure Gradient Flow Control by focusing on inter-cell cohesion rather than directional flow, and it is distinct from the Internal Pressure Controller which handles bulk effects like explosions [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of Surface cells, as it requires iterating over Surface cells to set pressure and compute gradients.
*   **Memory Complexity:** $O(N)$ for storing the pressure field on Surface cells.
*   **Strengths:** Direct control over surface flow direction; useful for artistic effects like guiding spray or sheeting.
*   **Limitations:** Does not model physical surface tension accurately; requires careful tuning to avoid instability.
*   **When to Use:** When artistic control over surface flow direction is needed, or to simulate effects like wind-driven spray.
*   **When Not to Use:** When physical accuracy of surface tension or bulk pressure effects is required.

## Background
Originated in the context of smoke and fluid animation, specifically in systems that use grid-based simulations with explicit pressure controllers [S1, unofficial]. It is part of a family of controllers including density, viscosity, and body force controllers, allowing animators to manipulate fluid properties directly.

## Sources
[S1] Efficient computation and rendering of smoke animation, http://hdl.handle.net/2346/16985, fetched 2026-10-26 (unofficial)
