# Fire Lights (GI Simulation)
Also called: Fire Light Simulation, Light Grid Lighting, emLightsCountEnbl
## Overview
Fire Lights simulates Global Illumination (GI) by placing light sources in the bright parts of a fire simulation, allowing the fire to illuminate scene objects and the simulator's own smoke without the performance cost of full GI calculation [S1]. It serves as a faster alternative to native GI, though it creates additional light sources that persist even if GI is enabled [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** Fire simulation data (emissive intensity/brightness per cell).
*   **Output:** A set of Omni light sources positioned within the fire volume, contributing to the scene's illumination.

**Data Structures:**
The method generates a collection of light objects. The position of each light corresponds to a "bright part" of the fire grid. The specific algorithm for selecting these parts is not detailed in the sources, but the lights are Omni lights [S1].

**Parameters:**
*   `emLightsCountEnbl` (Create Fire Lights): Boolean. When enabled, activates the simulation of GI via light placement [S1].
*   `emLightsPowerOnSelfMult` (Light Power on Self): Float. Multiplier for light intensity affecting the simulator's own smoke [S1].
*   `emLightsPowerMult` (Light Power on Scene): Float. Multiplier for light intensity affecting all scene objects except the simulator [S1].
*   `emLightsSelfShadow` (Self Shadowing): Enum. Controls smoke obstruction of light. Options: `None`, `Ray-traced`, `Grid-based` [S1].
*   `emLightsCaustSubdivs` (Caustic Subdivs): Integer. Sampling control for caustic effects [S1].
*   `emLightsPersist` (Create Lights Even If Not Renderable): Boolean. Forces light emission even if the simulator is not rendered [S1].

**Steps:**
1.  Analyze the fire simulation grid to identify bright regions [S1].
2.  Place Omni light sources in these bright regions [S1].
3.  Adjust the color and power of these lights automatically based on the fire's properties [S1].
4.  Apply intensity multipliers:
    *   Multiply intensity for self-illumination by `emLightsPowerOnSelfMult` [S1].
    *   Multiply intensity for scene illumination by `emLightsPowerMult` [S1].
5.  If `emLightsSelfShadow` is enabled:
    *   If `Ray-traced`: Perform ray tracing to determine if smoke obstructs the path from light sources [S1].
    *   If `Grid-based`: Use grid-based approximation to determine obstruction [S1].
    *   If `None`: No obstruction calculation; smoke is brightly lit [S1].
6.  If `emLightsCaustSubdivs` is active, use this value for caustic sampling [S1].
7.  If `emLightsPersist` is enabled, generate lights even if the fire simulator is disabled for rendering [S1].

**Edge Cases and Pitfalls:**
*   **V-Ray GPU:** All Fire Lights options are ignored when rendering with V-Ray GPU. GI must be enabled in V-Ray Settings for illumination in this mode [S1].
*   **GI Conflict:** If `emLightsCountEnbl` is enabled while rendering with GI, additional light sources are created on top of GI. This may cause double illumination or performance issues. Disable `emLightsCountEnbl` if natural GI illumination is desired [S1].
*   **V-Ray Object Properties:** The `Generate GI` and `Receive GI` options in V-Ray Object Properties are ignored while `emLightsCountEnbl` is enabled [S1].
*   **Performance:** Enabling Self Shadowing (especially Ray-traced) decreases rendering performance. Reducing Light Grid Resolution can help regain speed [S1].

## Subtopics

### Fire Lights
When enabled (`emLightsCountEnbl`), this feature forces the simulator to emit light over the scene even if the simulator itself is disabled for rendering. This is primarily used for compositing scenarios where the fire simulation is rendered in a separate pass [S1].

## Operating characteristics
*   **Performance:** Rendering is much faster than using Global Illumination (GI) [S1]. However, enabling Self Shadowing decreases performance, particularly with Ray-traced shadowing [S1].
*   **Quality:** Produces similar illumination results to GI but with less physical accuracy regarding light transport (e.g., indirect bounces are not simulated) [S1].
*   **Limitations:** Does not work with V-Ray GPU rendering [S1]. Can conflict with native GI if both are enabled [S1].
*   **Scaling:** Performance can be improved by reducing the Light Grid Resolution, which reduces the number of generated lights [S1].

## Background
This method is part of the Phoenix for Maya Voxel Shader, developed by Chaos Group. It provides a workaround for the high computational cost of Global Illumination in fire simulations, specifically tailored for the Phoenix Simulator environment [S1].

## Sources
[S1] Voxel Shader - Phoenix for Maya, https://documentation.chaos.com/space/PHX4MAYA/125768009, fetched 2026-10-29
