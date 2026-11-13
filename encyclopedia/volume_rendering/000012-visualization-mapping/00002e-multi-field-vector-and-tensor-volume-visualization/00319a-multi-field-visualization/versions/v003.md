# Multi Field Visualization

Also called: Multi Field Visualization DOP

## Overview
Multi Field Visualization is a DOP node that visualizes multiple field data, combining and ramping fields to preview rendering behavior [S1]. It differs from full volume rendering methods by operating as a fast geometry-based or simple volume proxy (height field or smoke) directly within the dynamics network, prioritizing speed and parameter tuning over photorealistic accuracy [S1].

## How it works
**Inputs and Outputs:**
The node accepts an optional object stream input (First Input) to specify which simulation objects are modified; objects connected here matching the Group parameter are modified [S1]. If unconnected, the node can be used with an Apply Data node or as input to another data node [S1]. Other data nodes attached to subsequent inputs act as modifiers for the data created by this node [S1]. The output is an object stream (if an object stream input is provided) containing the same objects with the visualization data attached, or a data output connectable to Apply Data DOP or other data nodes [S1].

**Data Structures:**
Not given in the sources.

**Numbered Steps:**
Not given in the sources.

**Formulas:**
Volume values $v$ are linearly remapped from a per-field mapping range $[R_{min}, R_{max}]$ into $v' \in [0, 1]$ using $v' = \frac{v - R_{min}}{R_{max} - R_{min}}$ [S1]. For the Emission Color Physical Blackbody mode, the temperature $T$ corresponding to a normalized field value is interpolated between $T_0$ (value 0.0) and $T_1$ (value 1.0), then scaled by $T_s$ before mapping to color via the blackbody radiation spectrum [S1].

**Parameters:**
*   **Field:** Specifies the name of the field to visualize. If a color field (diffuse/emission) is specified, a scalar field is treated as monochromatic. If a vector field is specified, its components are tied to the red, green, and blue channels [S1].
*   **Minimum, Maximum:** Defines the density volume range remapped to 0..1. It acts as the default range for all other volumes unless overridden [S1].
*   **Range:** Per-field mapping range $[R_{min}, R_{max}]$ [S1].
*   **Mode:** Controls ramp usage. 'No Ramp' uses values unclamped and unaffected. 'Clamped Ramp' clamps values to 0..1 before applying the ramp. 'Periodic Ramp' takes the modulus of values with 1 before sending to the ramp, useful for checkerboard or striped effects [S1].
*   **Ramp:** Volume values (after range conversion) are looked up in the ramp. If a color ramp has one volume bound, the value looks up the color. If two or three volumes are bound, each looks up R, G, B independently [S1].
*   **Emission Color Physical Blackbody:** Maps Emission Color Field values using the blackbody radiation spectrum, similar to the Physical (Black-Body) Color Mode on the Pyro Shader VOP [S1].
*   **Temperature Scale ($T_s$):** Scale applied to Emission Color Field values before mapping to color via blackbody spectrum [S1].
*   **Temperature at 0 (K) ($T_0$):** Sets the Kelvin temperature corresponding to a value of 0.0 in the Emission Color Field [S1].
*   **Temperature at 1 (K) ($T_1$):** Sets the Kelvin temperature corresponding to a value of 1.0 in the Emission Color Field [S1].
*   **Adaptation:** Manipulates the low end of the generated intensity [S1].
*   **Burn:** Manipulates the high end of the generated intensity [S1].
*   **Density Scale:** Fixed multiplier for the density field to control opacity [S1].
*   **Shadow Scale:** Multiplier applied to the Density Field after density scale, used for lighting. Equivalent to multiplying all light shadow intensities by this value [S1].
*   **Max Vis Res:** Default maximum volume resolution is 128 voxels per side to ensure fast shading and avoid texture memory issues [S1].
*   **Visualization Mode:** When set, 2D volumes are visualized as a height field instead of smoke [S1].
*   **Bounding Box:** Draws a bounding box encompassing the field [S1].
*   **Bounding Box Hash:** Adds hash marks along axes from the origin, marking voxel divisions. Marks are larger for every tenth and hundredth division [S1].

**Local Variables:**
The node defines local variables for each channel/parameter on the Data Options page, evaluating to the previous value stored in attached data [S1]. Key variables include:
*   `channelname`: Shortcut for `dopfield($DOPNET, $OBJID, dataName, "Options", 0, channelname)`. Returns 0 or empty string if data doesn't exist [S1].
*   `DATACT`/`DATACF`: Simulation time (ST) and frame (SF) at which current data was created [S1].
*   `ST`/`SF`: Simulation time and frame for node evaluation. `SF` equals `ST` divided by `TIMESTEP` [S1].
*   `TIMESTEP`: Size of a simulation timestep. `SFPS` is its inverse [S1].
*   `SNOBJ`/`NOBJ`: Total objects in simulation and objects evaluated by the current node, respectively [S1].
*   `OBJ`: Index of specific object being processed (0 to NOBJ-1). -1 if not processed sequentially [S1].
*   `OBJID`: Unique integer identifier for the object. Best way to look up info via `dopfield`. -1 if not processed sequentially [S1].
*   `ALLOBJIDS`/`ALLOBJNAMES`: Space-separated lists of unique object identifiers and names being processed [S1].
*   `OBJCT`/`OBJCF`: Simulation time and frame at which current object was created [S1].
*   `OBJNAME`: String name of object being processed. Empty string if not processed sequentially [S1].
*   `DOPNET`: Full path of the current DOP network [S1].
*   `RELNAME`, `RELOBJIDS`, `RELOBJNAMES`, `RELAFFOBJIDS`, `RELAFFOBJNAMES`: Set only when attaching data to a relationship, containing relationship name, affected object IDs/names, and affector object IDs/names [S1].

**Pitfalls:**
Fog density is per unit and independent of resolution [S1]. Small boxes become transparent, especially without HDR rendering [S1]. Enable HDR rendering in the Effects tab of the Display Options dialog to fix this [S1].

## Operating characteristics
*   **Performance:** Restricted to a default maximum of 128 voxels per side to ensure reasonably fast shading times and avoid texture memory problems [S1].
*   **Scaling:** Density is per unit, meaning opacity depends on scene scale rather than voxel count alone [S1].
*   **Limitations:** Provides a preview of rendering behavior rather than a final photorealistic render [S1].

## Background
Multi Field Visualization is a node within the Houdini Dynamics (DOP) network, introduced in version 12.0 [S1]. It is part of the suite of dynamics nodes designed to manipulate and visualize simulation data [S1].

## Sources
[S1] Multi Field Visualization - SideFX, https://www.sidefx.com/docs/houdini/nodes/dop/multifieldvisualization.html, fetched 2026-10-27
