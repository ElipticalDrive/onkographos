# Transfer Function Serialization

Also called: JSON Volume Preset Configuration, Volume Transfer Function Preset, bone.json/muscle.json preset, loadTransferFunction, saveTransferFunction

## Overview
Transfer Function Serialization converts an in-memory transfer function data structure into a persistent JSON format and reconstructs it upon loading [S1, S2, unofficial]. It solves the problem of preserving complex volume rendering configurations, including color/opacity controls and Gaussian objects, across application sessions [S1, S2, unofficial]. It is distinguished from its parent, the Transfer Function Core Data Structure, by its focus on I/O persistence rather than runtime evaluation [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A `TransferFunctionCore` object containing resolution, alpha arrays, color controls, opacity controls, and Gaussian objects [S1, unofficial].
*   **Output:** A JSON string or file representing the serialized state [S1, S2, unofficial].

**Data Structures:**
Two distinct JSON schemas are observed in the sources. Implementation must support both or select one based on the target ecosystem.

1.  **Schema A (C++/General):**
    *   `resolution`: `int`
    *   `alphaArray`: Object with `encoding` ("BASE64") and `data` (string) [S1, unofficial].
    *   `colorControls`: Array of objects `{ "position": float, "color": [float, float, float] }` [S1, unofficial].
    *   `opacityControl`: Array of objects `{ "position": float, "alpha": float }` (inferred as `vec2f` from S1 note, though S1 code implies separate control lists) [S1, unofficial].
    *   `gaussianObjects`: Array of objects `{ "mean": float, "sigma": float, "heightFactor": float }` [S1, unofficial].

2.  **Schema B (VTK/Python):**
    *   `color_transfer`: List of 4-element float arrays `[scalar, r, g, b]` [S2, unofficial].
    *   `scalar_opacity`: List of 2-element float arrays `[scalar, alpha]` [S2, unofficial].
    *   `gradient_opacity`: `None` or list [S2, unofficial].
    *   `specular`: `float` [S2, unofficial].
    *   `specular_power`: `int` [S2, unofficial].
    *   `ambient`: `float` [S2, unofficial].
    *   `diffuse`: `float` [S2, unofficial].
    *   `shade`: `bool` [S2, unofficial].
    *   `interpolation`: `bool` (e.g., "phong", "gouraud") [S2, unofficial].

**Numbered Steps:**

1.  **Serialize Color Controls:**
    *   For Schema A: Iterate through `colorControls`. For each control, extract `position` and `color` (vec3) [S1, unofficial].
    *   For Schema B: Construct a list of `[scalar, r, g, b]` arrays. Append `[0.5, 0.0]` to each entry as a trailing control or padding [S2, unofficial].

2.  **Serialize Opacity Controls:**
    *   For Schema A: Iterate through `opacityControl` (or `alphaControls`). Extract `position` and `alpha` [S1, unofficial].
    *   For Schema B: Construct a list of `[scalar, alpha]` arrays. Append `[0.25, 0.0]` to each entry [S2, unofficial].

3.  **Serialize Gaussian Objects:**
    *   For Schema A: Iterate through `gaussianObjects`. Extract `mean`, `sigma`, and `heightFactor` [S1, unofficial].
    *   For Schema B: Gaussian objects are not explicitly listed in the VTK preset structure; they may be handled via `scalar_opacity` or omitted [S2, unofficial].

4.  **Serialize Alpha Array:**
    *   For Schema A: Encode the `alphaArray` (typically a 1D or 2D float array) into Base64 [S1, unofficial].
    *   For Schema B: Not explicitly detailed in the VTK preset structure; may be implicit in the mapper state [S2, unofficial].

5.  **Serialize Lighting/Properties:**
    *   For Schema B: Include `specular`, `specular_power`, `ambient`, `diffuse`, `shade`, and `interpolation` [S2, unofficial].

6.  **Write JSON:**
    *   Serialize the resulting `OrderedDict` (or equivalent structure) to a JSON string with indentation for readability and persistence [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import json
import base64
import numpy as np

def serialize_tf_schema_a(tf_core):
    """
    Serializes TransferFunctionCore (Schema A) to JSON.
    Based on [S1, unofficial].
    """
    # 1. Handle Alpha Array
    alpha_data = tf_core.alpha_array # Assume numpy array or list
    alpha_bytes = alpha_data.tobytes()
    alpha_b64 = base64.b64encode(alpha_bytes).decode('utf-8')
    
    # 2. Handle Color Controls
    color_controls = []
    for ctrl in tf_core.color_controls:
        color_controls.append({
            "position": ctrl.position,
            "color": list(ctrl.color) # vec3f
        })
        
    # 3. Handle Opacity Controls
    opacity_controls = []
    for ctrl in tf_core.opacity_controls:
        opacity_controls.append({
            "position": ctrl.position,
            "alpha": ctrl.alpha
        })
        
    # 4. Handle Gaussian Objects
    gaussian_objects = []
    for gauss in tf_core.gaussian_objects:
        gaussian_objects.append({
            "mean": gauss.mean,
            "sigma": gauss.sigma,
            "heightFactor": gauss.height_factor
        })
        
    # 5. Construct Root Object
    preset = {
        "resolution": tf_core.resolution,
        "alphaArray": {
            "encoding": "BASE64",
            "data": alpha_b64
        },
        "colorControls": color_controls,
        "opacityControl": opacity_controls,
        "gaussianObjects": gaussian_objects
    }
    
    return json.dumps(preset, indent=2)

def serialize_tf_schema_b(vtk_actor):
    """
    Serializes VTK Volume Rendering properties (Schema B) to JSON.
    Based on [S2, unofficial].
    """
    # Extract properties from VTK actor/property
    prop = vtk_actor.GetProperty()
    mapper = vtk_actor.GetMapper()
    
    # Note: Extracting color/opacity ramps from VTK mapper is complex.
    # This pseudocode assumes helper functions exist to extract the ramps
    # as lists of [scalar, r, g, b] and [scalar, alpha].
    
    color_transfer = extract_color_ramp(mapper) # List of [s, r, g, b]
    scalar_opacity = extract_opacity_ramp(mapper) # List of [s, alpha]
    
    # Append trailing controls as per [S2, unofficial]
    for i in range(len(color_transfer)):
        color_transfer[i].append(0.5)
        color_transfer[i].append(0.0)
        
    for i in range(len(scalar_opacity)):
        scalar_opacity[i].append(0.25)
        scalar_opacity[i].append(0.0)
        
    preset = {
        "color_transfer": color_transfer,
        "scalar_opacity": scalar_opacity,
        "gradient_opacity": None,
        "specular": prop.GetSpecular(),
        "specular_power": int(prop.GetSpecularPower()),
        "ambient": prop.GetAmbient(),
        "diffuse": prop.GetDiffuse(),
        "shade": prop.ShadeOn(),
        "interpolation": "phong" if prop.InterpolationIsPhong() else "gouraud"
    }
    
    return json.dumps(preset, indent=2)

def deserialize_tf_schema_a(json_str):
    """
    Deserializes JSON (Schema A) to TransferFunctionCore.
    Based on [S1, unofficial].
    """
    preset = json.loads(json_str)
    
    # 1. Alpha Array
    alpha_b64 = preset["alphaArray"]["data"]
    alpha_bytes = base64.b64decode(alpha_b64)
    # Assume resolution is known or stored in preset
    resolution = preset["resolution"]
    alpha_array = np.frombuffer(alpha_bytes, dtype=np.float32).reshape(resolution)
    
    # 2. Color Controls
    color_controls = []
    for ctrl in preset.get("colorControls", []):
        # Pitfall: Missing keys are silently skipped [S1, unofficial]
        if "position" not in ctrl or "color" not in ctrl:
            continue
        color_controls.append({
            "position": ctrl["position"],
            "color": ctrl["color"]
        })
        
    # 3. Opacity Controls
    opacity_controls = []
    for ctrl in preset.get("opacityControl", []):
        if "position" not in ctrl or "alpha" not in ctrl:
            continue
        opacity_controls.append({
            "position": ctrl["position"],
            "alpha": ctrl["alpha"]
        })
        
    # 4. Gaussian Objects
    gaussian_objects = []
    for gauss in preset.get("gaussianObjects", []):
        # Pitfall: Missing keys are silently skipped [S1, unofficial]
        if "mean" not in gauss or "sigma" not in gauss or "heightFactor" not in gauss:
            continue
        gaussian_objects.append({
            "mean": gauss["mean"],
            "sigma": gauss["sigma"],
            "heightFactor": gauss["heightFactor"]
        })
        
    # Reconstruct Core Object
    tf_core = TransferFunctionCore()
    tf_core.resolution = resolution
    tf_core.alpha_array = alpha_array
    tf_core.color_controls = color_controls
    tf_core.opacity_controls = opacity_controls
    tf_core.gaussian_objects = gaussian_objects
    
    return tf_core
```

**Parameters:**
*   Indentation: Standard JSON pretty-print (2 or 4 spaces) [S2, unofficial].
*   Base64 Encoding: Standard RFC 4648 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Missing Keys:** If `position` or `color` is missing in `colorControls`, the control is skipped [S1, unofficial]. If `mean`, `sigma`, or `heightFactor` is missing in `gaussianObjects`, it is skipped [S1, unofficial].
*   **Trailing Controls:** Schema B requires appending `[0.5, 0.0]` to color entries and `[0.25, 0.0]` to opacity entries [S2, unofficial].
*   **Alpha Array Size:** Gaussian objects may require the alpha array size to match the resolution [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of controls and the size of the alpha array (for Base64 encoding) [S1, S2, unofficial].
*   **Memory Complexity:** Proportional to the size of the JSON string, which is approximately 1.33x the size of the binary alpha array plus overhead for control definitions [S1, unofficial].
*   **Strengths:** Human-readable format; supports complex configurations including Gaussian objects and lighting parameters [S1, S2, unofficial].
*   **Limitations:** Base64 encoding increases file size; parsing overhead for large alpha arrays [S1, unofficial].
*   **When to use:** When persistence of volume rendering presets is required [S1, S2, unofficial].
*   **When not to use:** When real-time performance is critical and the transfer function is simple enough to be hardcoded or computed on the fly [S2, unofficial].

## Background
Transfer function serialization is a common practice in volume rendering applications to allow users to save and load custom transfer functions [S1, S2, unofficial]. The method is implemented in various projects, including the TransferFunctionModule (C++) and VTK-based volume renderers (Python) [S1, S2, unofficial].

## Sources
[S1] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
[S2] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
