# Multi Field Visualization

Also called: Multi Field Visualization DOP

## Overview
Multi Field Visualization is a diagnostic and preview tool for visualizing multiple scalar, vector, and color fields within a simulation environment. It combines and ramps field data to provide a quick preview of rendering behavior, allowing users to inspect density, emission, and vector attributes without the computational cost of full ray-traced volume rendering [S1]. It differs from full rendering methods by operating as a geometry-based or simple volume proxy (height field or smoke) directly within the dynamics network, prioritizing speed and parameter tuning over photorealistic accuracy [S1].

## How it works
**Inputs and Outputs:**
The node accepts an optional object stream input (First Input) to specify which simulation objects are modified. If unconnected, it can be used with an Apply Data node or as input to another data node. Other data nodes attached to subsequent inputs act as modifiers for the data created by this node [S1]. The output is an object stream (if an object stream input is provided) containing the same objects with the visualization data attached, or a data output connectable to Apply Data DOP or other data nodes [S1].

**Data Structures and Parameters:**
The visualization relies on several key parameters to map field values to visual attributes:
*   **Field:** Specifies the name of the field to visualize. Scalar fields are treated as monochromatic for color fields. Vector fields are tied to RGB channels [S1].
*   **Range:** A per-field mapping range $[R_{min}, R_{max}]$. Volume values $v$ are linearly remapped to $v' \in [0, 1]$ using:
    $$ v' = \text{clamp}\left(\frac{v - R_{min}}{R_{max} - R_{min}}, 0, 1\right) $$
    If not specified, the Density volume's Minimum/Maximum range is used as the default [S1].
*   **Mode:** Controls ramp application:
    *   *No Ramp:* Values are used unclamped and unaffected by the ramp [S1].
    *   *Clamped Ramp:* Values are clamped to $[0, 1]$ then the ramp is applied [S1].
    *   *Periodic Ramp:* The modulus of values with 1 is sent to the ramp ($v' \mod 1$), useful for checkerboard/striped effects [S1].
*   **Ramp:** A lookup table. If a color ramp has one volume bound, the value looks up the color. If two or three volumes are bound, each looks up R, G, B independently [S1].
*   **Emission Color Physical Blackbody:** Maps Emission Color Field values using the blackbody radiation spectrum. Requires **Temperature Scale** ($T_s$), **Temperature at 0 (K)** ($T_0$), and **Temperature at 1 (K)** ($T_1$). The temperature $T$ for a normalized value $v'$ is interpolated:
    $$ T = T_0 + v' \times (T_1 - T_0) $$
    This temperature is then mapped to color via the Planck radiation law approximation [S1].
*   **Adaptation/Burn:** Manipulate the low and high ends of the generated intensity, respectively [S1].
*   **Density Scale:** Fixed multiplier for the density field to control opacity [S1].
*   **Shadow Scale:** Multiplier applied to the Density Field after density scale, used for lighting [S1].
*   **Max Vis Res:** Default maximum volume resolution is 128 voxels per side [S1].
*   **Visualization Mode:** 2D volumes can be visualized as a height field instead of smoke [S1].
*   **Bounding Box/Bounding Box Hash:** Draws a bounding box and hash marks marking voxel divisions (larger marks for every 10th and 100th division) [S1].

**Local Variables:**
The node defines local variables for each channel/parameter on the Data Options page, evaluating to the previous value stored in attached data. Key variables include:
*   `channelname`: Shortcut for `dopfield($DOPNET, $OBJID, dataName, "Options", 0, channelname)`. Returns 0 or empty string if data doesn't exist [S1].
*   `DATACT`/`DATACF`: Simulation time/frame at which current data was created [S1].
*   `ST`/`SF`: Simulation time/frame for node evaluation. `SF = ST / TIMESTEP` [S1].
*   `OBJID`: Unique integer identifier for the object. Best way to look up info via `dopfield`. -1 if not processed sequentially [S1].
*   `OBJ`: Index of specific object being processed (0 to NOBJ-1). -1 if not processed sequentially [S1].
*   `DOPNET`: Full path of the current DOP network [S1].

**Pitfalls:**
Fog density is per unit and independent of resolution. Small boxes become transparent, especially without HDR rendering. Enable HDR rendering in the Effects tab of the Display Options dialog to fix [S1].

## Operating characteristics
*   **Performance:** Restricted to a default maximum of 128 voxels per side to ensure fast shading and avoid texture memory issues [S1].
*   **Accuracy:** Provides a preview of rendering behavior, not a final render. Density is per unit, meaning opacity depends on scene scale, not just voxel count [S1].
*   **Usage:** Best for debugging field values, tuning transfer functions, and checking vector field directions during simulation development [S1].

## Background
Multi Field Visualization is a node within the Houdini Dynamics (DOP) network, introduced in version 12.0 [S1]. It is part of the suite of dynamics nodes designed to manipulate and visualize simulation data [S1].

## Sources
[S1] Multi Field Visualization - SideFX, https://www.sidefx.com/docs/houdini/nodes/dop/multifieldvisualization.html, fetched 2026-10-27
