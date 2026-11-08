# AI-Based Transfer Function Peak Prediction

Also called: AI-Assisted Transfer Function Selection, ONNX Transfer Function Picker, Peak Predictor, usePeakPredictor, usePeakPredictor hook

## Overview
This method automates the selection of the `peak` parameter for a volume rendering transfer function by analyzing rendered frames using a lightweight neural network [S1, unofficial]. It solves the tedious manual tuning required when browsing many volume/surface pairs, particularly in fluorescence microscopy data where the optimal opacity peak varies significantly [S1, unofficial]. Unlike general transfer function optimization methods that adjust full RGBA maps, this approach specifically targets the single scalar `peak` parameter of a tent-shaped alpha function [S1, unofficial].

## How it works
The method operates as a React hook (`usePeakPredictor`) that intercepts the rendering pipeline to capture frames, process them into a tensor, and query an ONNX model for the optimal peak value [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Rendered pixel buffers (RGB) and surface masks from the renderer.
*   **Output:** A scalar `peak` value (integer 0–255) applied to the transfer function.

**Data Structures:**
1.  **Transfer Function Texture:** Implemented as a Three.js `DataTexture` with width 256, height 1, and `THREE.RGBAFormat` [S1, unofficial].
2.  **Input Tensor:** A 192x192 multi-channel tensor constructed from:
    *   Two binary masks: Dilation of the green surface mask and a crop region mask.
    *   The red channel of the combined render frame [S1, unofficial].
3.  **Neural Network:** MobileNetV3-Small model exported to ONNX format [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Load the ONNX Runtime Web library (version ^1.24.3) and the MobileNetV3-Small model [S1, unofficial]. If the library is missing, the build fails with a Webpack resolution error [S1, unofficial]. The WASM binary is fetched from a CDN; if it does not respond within 10 seconds, an exception is thrown [S1, unofficial].
2.  **Capture:** Cycle through a range of transfer functions (high peak to low peak) to generate candidate renders [S1, unofficial].
3.  **Buffer Reading:** Use `gl.readPixels` to capture the resulting pixel buffers for each candidate [S1, unofficial].
4.  **Verification:** To ensure frame integrity, a fragment shader writes a one-byte watermark into the blue channel of the image center pixel. The predictor reads this byte back and compares it against the expected value before accepting the frame [S1, unofficial].
5.  **Surface Capture:** Capture a separate surface-only frame to help the model locate the green surface within the image [S1, unofficial].
6.  **Tensor Construction:**
    *   Crop and resize the captured data to 192x192 pixels [S1, unofficial].
    *   Construct the multi-channel tensor from the binary masks and the red channel of the render [S1, unofficial].
7.  **Prediction:** Feed the tensor into the ONNX model [S1, unofficial].
8.  **Application:** Apply the predicted peak value to the transfer function [S1, unofficial].

**Transfer Function Definition:**
The transfer function uses a tent-shaped alpha curve defined by `makeFluoTransferTex(alpha0, peak, dataGamma, alpha1, colorStr)` [S1, unofficial].
*   **Alpha Curve:** Alpha rises from `alpha0` at data value 0 to 255 at `peak`, then falls to `alpha1` at data value 255 [S1, unofficial].
*   **Gamma Correction:** The rise and fall shape is controlled by `dataGamma` using the power function $y^{(1.0 / \text{dataGamma})}$, where $y$ is the normalized data value [S1, unofficial]. Typical `dataGamma` is 0.5 [S1, unofficial].
*   **Final Gamma:** After ray casting accumulation, a `finalGamma` (typically 4.5) is applied to the accumulated color $c$ via $c^{(1.0 / \text{finalGamma})}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import onnxruntime_web
import gl_matrix

class PeakPredictor:
    def __init__(self, model_path, renderer):
        self.renderer = renderer
        self.model = onnxruntime_web.InferenceSession(model_path)
        self.expected_watermark = 0xAA
        
    def predict_peak(self):
        # 1. Capture candidate frames
        candidates = []
        for peak_val in range(0, 256, 10):
            self.renderer.set_transfer_function_peak(peak_val)
            self.renderer.render()
            
            # 2. Read pixels
            pixels = gl_matrix.read_pixels()
            
            # 3. Verify watermark
            center_pixel = pixels[center_x, center_y]
            if center_pixel.blue != self.expected_watermark:
                continue # Skip invalid frame
                
            candidates.append(pixels)
            
        # 4. Process tensor
        # Capture surface mask separately
        surface_mask = self.renderer.render_surface_only()
        
        # Construct tensor: [dilated_surface, crop_mask, render_red_channel]
        # Resize to 192x192
        tensor = self._build_tensor(candidates, surface_mask)
        
        # 5. Run model
        input_name = self.model.get_inputs()[0].name
        outputs = self.model.run(None, {input_name: tensor})
        
        # 6. Return predicted peak
        return outputs[0]

    def _build_tensor(self, frames, surface_mask):
        # Implementation details for cropping, resizing, and channel stacking
        # Not given in the sources.
        pass
```

## Operating characteristics
*   **Performance:** The method introduces latency due to multiple render passes (cycling through peaks) and the overhead of `gl.readPixels` and WASM inference [S1, unofficial].
*   **Hardware:** Requires a GPU capable of WebGL rendering and a CPU capable of running WebAssembly [S1, unofficial].
*   **Strengths:** Eliminates manual tuning for fluorescence microscopy data; integrates directly into web-based viewers [S1, unofficial].
*   **Limitations:** Specific to the tent-shaped transfer function described; requires the ONNX Runtime Web peer dependency; sensitive to CDN availability for WASM loading [S1, unofficial].
*   **When to use:** When browsing large collections of volume data with varying intensity distributions where manual peak adjustment is prohibitive [S1, unofficial].

## Background
The method is implemented in the `web-vol-viewer` project by Janelia Science Computing [S1, unofficial]. It is designed for fluorescence microscopy data, building on the transfer function principles described in Wan et al., "FluoRender: An Application of 2D Image Space Methods for 3D and 4D Confocal Microscopy Data Visualization in Neurobiology Research" (2012) [S1, unofficial]. The desktop application VVD Viewer also utilizes this transfer function [S1, unofficial].

## Sources
[S1] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
