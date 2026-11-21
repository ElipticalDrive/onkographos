# Transfer Function Serialization

Also called: JSON Volume Preset Configuration, Volume Transfer Function Preset, bone.json/muscle.json preset, loadTransferFunction, saveTransferFunction

## Overview
Transfer Function Serialization converts an in-memory transfer function data structure into a persistent JSON format and reconstructs it upon loading [S1, unofficial][S2, unofficial]. It stores volume rendering transfer function parameters, including color and opacity controls, Gaussian objects, and lighting settings, in a JSON file for runtime loading [S2, unofficial]. It is distinguished from its parent, the Transfer Function Core Data Structure, by its focus on I/O persistence rather than runtime evaluation [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A transfer function data structure containing resolution, alpha arrays, color controls, opacity controls, and Gaussian objects [S1, unofficial].
*   **Output:** A JSON string or file representing the serialized state.

**Data Structures:**
Two distinct JSON schemas are observed in the sources [S1, unofficial][S2, unofficial].

1.  **Schema A (C++/General):**
    *   `resolution`: `int`
    *   `alphaArray`: Object with `encoding` (`"BASE64"`) and `data` (string) [S1, unofficial].
    *   `colorControls`: Array of objects `{ "position": float, "color": vec3f }` [S1, unofficial].
    *   `opacityControl`: Array of objects `{ "position": vec2f }` [S1, unofficial].
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
    *   `interpolation`: `bool` or string (e.g., `"phong"`, `"gouraud"`) [S2, unofficial].

**Numbered Steps:**

1.  **Serialize Color Controls:**
    *   For Schema A: Iterate through `colorControls` and store `position` and `color` (vec3) [S1, unofficial].
    *   For Schema B: Define color transfer as a list of `[scalar, r, g, b]` arrays, then append `[0.5, 0.0]` to each entry [S2, unofficial].
2.  **Serialize Opacity Controls:**
    *   For Schema A: Iterate through `opacityControl` and store `position` and `alpha` [S1, unofficial].
    *   For Schema B: Define scalar opacity as a list of `[scalar, alpha]` arrays, then append `[0.25, 0.0]` to each entry [S2, unofficial].
3.  **Serialize Gaussian Objects:**
    *   For Schema A: Iterate through `gaussianObjects` and store `mean`, `sigma`, and `heightFactor` [S1, unofficial].
    *   For Schema B: Gaussian objects are not explicitly listed in the VTK preset structure [S2, unofficial].
4.  **Serialize Alpha Array:**
    *   For Schema A: Encode the `alphaArray` into Base64 within an object specifying `encoding: "BASE64"` and `data` [S1, unofficial].
    *   For Schema B: Alpha array serialization is not explicitly detailed in the VTK preset structure [S2, unofficial].
5.  **Serialize Lighting/Properties:**
    *   For Schema B: Include `specular`, `specular_power`, `ambient`, `diffuse`, `shade`, and `interpolation` [S2, unofficial].
6.  **Write JSON:**
    *   Serialize the resulting `OrderedDict` (or equivalent structure) to a JSON string with indentation for persistence [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import json
import base64

def serialize_transfer_function(tf_data, schema="A"):
    """
    Serializes a transfer function to JSON based on the target schema.
    Schema A matches the C++ TransferFunctionCore structure [S1, unofficial].
    Schema B matches the VTK Python preset structure [S2, unofficial].
    """
    if schema == "A":
        # 1. Alpha Array: Encode to Base64 [S1, unofficial]
        alpha_bytes = tf_data["alpha_array"] if isinstance(tf_data["alpha_array"], bytes) else bytes(tf_data["alpha_array"])
        alpha_b64 = base64.b64encode(alpha_bytes).decode("utf-8")

        # 2. Color Controls: Store position and color (vec3) [S1, unofficial]
        color_controls = [
            {"position": c["position"], "color": list(c["color"])}
            for c in tf_data.get("color_controls", [])
        ]

        # 3. Opacity Controls: Store position and alpha [S1, unofficial]
        opacity_controls = [
            {"position": c["position"], "alpha": c["alpha"]}
            for c in tf_data.get("opacity_controls", [])
        ]

        # 4. Gaussian Objects: Store mean, sigma, heightFactor [S1, unofficial]
        gaussian_objects = [
            {"mean": g["mean"], "sigma": g["sigma"], "heightFactor": g["heightFactor"]}
            for g in tf_data.get("gaussian_objects", [])
        ]

        preset = {
            "resolution": tf_data["resolution"],
            "alphaArray": {"encoding": "BASE64", "data": alpha_b64},
            "colorControls": color_controls,
            "opacityControl": opacity_controls,
            "gaussianObjects": gaussian_objects
        }

    elif schema == "B":
        # 1. Color Transfer: List of [scalar, r, g, b], append [0.5, 0.0] to each entry [S2, unofficial]
        color_transfer = [list(entry) + [0.5, 0.0] for entry in tf_data.get("color_transfer", [])]

        # 2. Scalar Opacity: List of [scalar, alpha], append [0.25, 0.0] to each entry [S2, unofficial]
        scalar_opacity = [list(entry) + [0.25, 0.0] for entry in tf_data.get("scalar_opacity", [])]

        # 3. Lighting/Properties [S2, unofficial]
        preset = {
            "color_transfer": color_transfer,
            "scalar_opacity": scalar_opacity,
            "gradient_opacity": None,
            "specular": tf_data.get("specular", 0.0),
            "specular_power": int(tf_data.get("specular_power", 1)),
            "ambient": tf_data.get("ambient", 0.3),
            "diffuse": tf_data.get("diffuse", 0.6),
            "shade": tf_data.get("shade", True),
            "interpolation": tf_data.get("interpolation", "phong")
        }

    # 4. Write JSON with indentation for persistence [S2, unofficial]
    return json.dumps(preset, indent=2)

def deserialize_transfer_function(json_str, schema="A"):
    """
    Deserializes JSON to a transfer function data structure.
    Handles missing keys by skipping invalid entries [S1, unofficial].
    """
    preset = json.loads(json_str)

    if schema == "A":
        # 1. Alpha Array: Decode Base64 [S1, unofficial]
        alpha_bytes = base64.b64decode(preset["alphaArray"]["data"])
        alpha_array = bytearray(alpha_bytes)

        # 2. Color Controls: Skip if position or color missing [S1, unofficial]
        color_controls = [
            c for c in preset.get("colorControls", [])
            if "position" in c and "color" in c
        ]

        # 3. Opacity Controls: Skip if position or alpha missing
        opacity_controls = [
            c for c in preset.get("opacityControl", [])
            if "position" in c and "alpha" in c
        ]

        # 4. Gaussian Objects: Skip if mean, sigma, or heightFactor missing [S1, unofficial]
        gaussian_objects = [
            g for g in preset.get("gaussianObjects", [])
            if "mean" in g and "sigma" in g and "heightFactor" in g
        ]

        return {
            "resolution": preset["resolution"],
            "alpha_array": alpha_array,
            "color_controls": color_controls,
            "opacity_controls": opacity_controls,
            "gaussian_objects": gaussian_objects
        }

    elif schema == "B":
        # Schema B reconstruction is straightforward list/dict mapping [S2, unofficial]
        return {
            "color_transfer": preset.get("color_transfer", []),
            "scalar_opacity": preset.get("scalar_opacity", []),
            "gradient_opacity": preset.get("gradient_opacity"),
            "specular": preset.get("specular", 0.0),
            "specular_power": preset.get("specular_power", 1),
            "ambient": preset.get("ambient", 0.3),
            "diffuse": preset.get("diffuse", 0.6),
            "shade": preset.get("shade", True),
            "interpolation": preset.get("interpolation", "phong")
        }
```

**Parameters:**
*   Indentation: Standard JSON pretty-print for persistence [S2, unofficial].
*   Base64 Encoding: Standard encoding for the alpha array [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Missing Keys:** If `position` or `color` is missing in `colorControls`, the control is skipped [S1, unofficial]. If `mean`, `sigma`, or `heightFactor` is missing in `gaussianObjects`, it is skipped [S1, unofficial].
*   **Trailing Controls:** Schema B requires appending `[0.5, 0.0]` to color entries and `[0.25, 0.0]` to opacity entries [S2, unofficial].
*   **Alpha Array Size:** Gaussian objects may require the alpha array size to match the resolution [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [G].
*   **Memory Complexity:** Not given in the sources [G].
*   **Strengths:** Human-readable format; supports complex configurations including Gaussian objects and lighting parameters [S1, unofficial][S2, unofficial].
*   **Limitations:** Base64 encoding increases file size; parsing overhead for large alpha arrays [S1, unofficial].
*   **When to use:** Not given in the sources [G].
*   **When not to use:** Not given in the sources [G].

## Background
Transfer function serialization is a common practice in volume rendering applications to allow users to save and load custom transfer functions [S1, unofficial][S2, unofficial].

## Sources
[S1] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
[S2] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
