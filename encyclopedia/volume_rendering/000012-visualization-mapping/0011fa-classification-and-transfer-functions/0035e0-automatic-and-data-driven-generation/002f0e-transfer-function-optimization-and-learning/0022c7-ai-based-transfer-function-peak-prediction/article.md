# AI-Based Transfer Function Peak Prediction

Also called: AI-Assisted Transfer Function Selection, ONNX Transfer Function Picker, Peak Predictor, usePeakPredictor, usePeakPredictor hook

## Overview
This method automates the selection of the `peak` parameter for a fluorescence microscopy volume rendering transfer function by analyzing rendered frames using a lightweight neural network [S1, unofficial]. It solves the tedious manual tuning required when browsing many volume/surface pairs by predicting the optimal opacity peak that best highlights the surface of interest [S1, unofficial]. It is distinct from general transfer function optimization methods by focusing specifically on a single scalar parameter (`peak`) within a predefined fluorescence transfer function shape, using image-space features rather than volume-space data properties [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A WebGL renderer context, a current volume rendering configuration, and a target surface color (typically green).
- **Output:** A predicted `peak` value (integer 0–255) applied to the transfer function.

**Data Structures:**
1.  **Transfer Function Texture:** A `THREE.DataTexture` with width 256, height 1, and format `THREE.RGBAFormat`. Each 8-bit data value maps to a 4-byte RGBA pixel. The alpha channel follows a "tent" shape:
    -   Rises from `alpha0` at data value 0 to 255 at `peak`.
    -   Falls to `alpha1` at data value 255.
    -   The rise/fall is controlled by `dataGamma` via $y^{(1.0 / \text{dataGamma})}$, where $y$ is the normalized data value [S1, unofficial].
2.  **Input Tensor:** A multi-channel tensor of size $192 \times 192$ pixels.
3.  **ONNX Model:** MobileNetV3-Small, running in WebAssembly via ONNX Runtime Web [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** Load the ONNX Runtime Web library. If the CDN fails to respond within 10 seconds, throw an exception [S1, unofficial].
2.  **Capture Loop:** Cycle through a range of `peak` values (e.g., high to low) to generate candidate renders.
3.  **Frame Capture:** For each candidate `peak`:
    a. Render the volume with the current `peak` value.
    b. Capture the pixel buffer using `gl.readPixels`.
    c. Render a surface-only frame to isolate the target surface (green) [S1, unofficial].
4.  **Verification:** Write a one-byte watermark into the blue channel of the center pixel of the rendered image. Read back the byte and compare against the expected value. Discard frames that fail verification to ensure rendering integrity [S1, unofficial].
5.  **Tensor Construction:**
    a. Create two binary masks: one from the dilation of the green surface, and one for the crop region.
    b. Extract the red channel from the combined render frame.
    c. Combine these channels into a single tensor.
    d. Crop and resize the tensor to $192 \times 192$ pixels [S1, unofficial].
6.  **Prediction:** Feed the tensor into the MobileNetV3-Small ONNX model.
7.  **Application:** Apply the predicted `peak` value to the transfer function texture [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import onnxruntime_web as ort
import numpy as np

class PeakPredictor:
    def __init__(self, model_path, gl_context):
        self.gl = gl_context
        # Load ONNX model in WASM
        self.session = ort.InferenceSession(model_path)
        self.target_color = (0, 255, 0) # Green

    def predict_peak(self, volume_data, surface_mesh):
        # Step 1: Generate candidate renders and capture
        candidates = []
        for peak in range(0, 256, 10): # Sample range
            # Set transfer function peak
            tf_texture = make_fluo_transfer_tex(alpha0=0, peak=peak, dataGamma=0.5, alpha1=255, colorStr='#ff00ff')
            
            # Render volume
            self.gl.render_volume(volume_data, tf_texture)
            frame = self.gl.read_pixels()
            
            # Verification watermark
            expected_watermark = 0xAA
            self.gl.write_pixel_center(frame, channel='blue', value=expected_watermark)
            actual_watermark = self.gl.read_pixel_center(frame, channel='blue')
            
            if actual_watermark != expected_watermark:
                continue # Skip invalid frame
            
            # Render surface only for mask
            self.gl.render_surface(surface_mesh, color=self.target_color)
            surface_frame = self.gl.read_pixels()
            
            candidates.append({
                'peak': peak,
                'frame': frame,
                'surface': surface_frame
            })
        
        if not candidates:
            return 128 # Fallback

        # Step 2: Build tensor from best candidate or aggregate
        # Note: The source implies feeding the tensor to the model. 
        # It does not specify if multiple frames are aggregated or if one is selected.
        # We assume the model takes a single frame representation.
        # Selecting the first valid candidate for demonstration, 
        # though in practice, the model might be trained on specific conditions.
        candidate = candidates[0]
        
        # Create masks
        green_mask = self.create_binary_mask(candidate['surface'], self.target_color)
        dilated_mask = self.dilate(green_mask, radius=2)
        crop_mask = self.get_crop_region()
        
        # Extract red channel from volume render
        red_channel = candidate['frame'][:, :, 0]
        
        # Combine channels: [dilated_mask, crop_mask, red_channel]
        # Assuming 3 channels for the tensor input
        tensor_input = np.stack([dilated_mask, crop_mask, red_channel], axis=-1)
        
        # Resize to 192x192
        tensor_input = self.resize(tensor_input, (192, 192))
        
        # Normalize if required by model (not specified, assuming raw 0-255 or 0-1)
        # [G] ONNX models often expect float32 normalized to [0,1] or [-1,1].
        # Assuming standard ImageNet normalization or raw byte scaling.
        # Since MobileNetV3 usually expects normalized inputs, we scale to [0,1].
        tensor_input = tensor_input.astype(np.float32) / 255.0
        
        # Step 3: Predict
        # ONNX Runtime Web expects input as a specific format
        input_name = self.session.get_inputs()[0].name
        outputs = self.session.run(None, {input_name: tensor_input})
        
        # Output is the predicted peak value
        predicted_peak = int(outputs[0][0][0])
        
        return predicted_peak

    def make_fluo_transfer_tex(self, alpha0, peak, dataGamma, alpha1, colorStr):
        # Implementation of the tent-shaped alpha function
        # y = normalized data value [0,1]
        # alpha = alpha0 + (255 - alpha0) * (y ** (1/dataGamma)) for y <= peak/255
        # alpha = 255 - (255 - alpha1) * ((1-y) ** (1/dataGamma)) for y > peak/255
        pass

    def create_binary_mask(self, frame, color):
        # Return 1 where pixel matches color, 0 otherwise
        pass

    def dilate(self, mask, radius):
        # Morphological dilation
        pass

    def get_crop_region(self):
        # Return binary mask of the region of interest
        pass

    def resize(self, image, size):
        # Resize image to size
        pass
```

**Parameters:**
-   `alpha0`: 0 (typical)
-   `alpha1`: 255 (typical)
-   `dataGamma`: 0.5 (typical)
-   `finalGamma`: 4.5 (applied after ray casting)
-   Model: MobileNetV3-Small
-   Input Tensor Size: $192 \times 192$
-   ONNX Runtime Web Version: ^1.24.3 [S1, unofficial]

**Edge Cases and Pitfalls:**
-   **Dependency Failure:** If `onnxruntime-web` is not installed, the build fails with a Webpack error: 'Module not found: Error: Can't resolve 'onnxruntime-web'...' [S1, unofficial].
-   **Network Timeout:** If the CDN hosting the WASM code does not respond within 10 seconds, an exception is thrown [S1, unofficial].
-   **Verification Failure:** If the watermark check fails, the frame is discarded. This may happen if the renderer does not support precise pixel writes or reads [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Depends on the number of candidate renders generated and the inference time of MobileNetV3-Small. The capture loop runs $O(N)$ times where $N$ is the number of sampled peaks. Inference is $O(1)$ relative to volume size but depends on tensor size ($192 \times 192$).
-   **Memory Complexity:** $O(W \times H)$ for frame buffers and tensor storage.
-   **Strengths:** Automates tedious manual tuning for fluorescence microscopy data. Lightweight model suitable for web environments.
-   **Limitations:** Requires WebGL access and specific rendering pipeline modifications (watermarking). Dependent on CDN availability. Only predicts the `peak` parameter, not the full transfer function.
-   **When to use:** When browsing many volume/surface pairs in a web-based viewer where manual tuning is inefficient.
-   **When not to use:** When full transfer function optimization is needed, or when network access is restricted.

## Background
This method is implemented in the `web-vol-viewer` project by JaneliaSciComp [S1, unofficial]. It is designed for fluorescence microscopy data visualization, building on the FluoRender transfer function described by Wan et al. (2012) [S1, unofficial]. The use of ONNX Runtime Web allows the neural network to run in the browser via WebAssembly, enabling client-side prediction without server round-trips.

## Sources
[S1] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
