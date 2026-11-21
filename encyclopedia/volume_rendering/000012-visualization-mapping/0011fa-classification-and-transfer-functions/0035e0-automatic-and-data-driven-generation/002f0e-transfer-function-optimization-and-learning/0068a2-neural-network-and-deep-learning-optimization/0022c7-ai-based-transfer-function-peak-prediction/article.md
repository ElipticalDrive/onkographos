# AI-Based Transfer Function Peak Prediction

Also called: AI-Assisted Transfer Function Selection, ONNX Transfer Function Picker, Peak Predictor, usePeakPredictor, usePeakPredictor hook

## Overview
This method automates the selection of the `peak` parameter for a volume rendering transfer function by analyzing rendered frames using a neural network. It solves the problem of tedious manual parameter tuning when browsing many volume/surface pairs [S1, unofficial]. It is distinct from gradient-based optimization methods (e.g., [Transfer Function Reconstruction Loss](1199-transfer-function-reconstruction-loss.md)) because it operates in image space using a pre-trained model rather than differentiable volume rendering or loss minimization.

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of rendered frames corresponding to different transfer function configurations, specifically varying the `peak` parameter.
*   **Output:** A single scalar value representing the predicted optimal `peak` parameter for the current volume data.

**Data Structures:**
The transfer function is implemented as a `DataTexture` with width 256, height 1, and `RGBAFormat` (4 bytes per 8-bit data value) [S1, unofficial]. The alpha channel follows a "tent" shape: rising from `alpha0` at data value 0 to 255 at `peak`, then falling to `alpha1` at data value 255 [S1, unofficial]. The rise/fall shape is controlled by `dataGamma` via $y^{(1.0 / \text{dataGamma})}$, where $y$ is the normalized data value [S1, unofficial].

**Steps:**
1.  **Initialization:** Load the ONNX Runtime Web environment. If the CDN fails to respond within 10 seconds, throw an exception [S1, unofficial].
2.  **Sampling:** Cycle through a range of `peak` values (high to low). For each `peak`:
    a. Apply the transfer function with the current `peak`.
    b. Render the volume.
    c. Capture the pixel buffer using `gl.readPixels`.
    d. Capture a surface-only frame to locate the green surface [S1, unofficial].
3.  **Tensor Construction:** Build a multi-channel tensor from the captures:
    a. Create two binary masks: one from the dilation of the green surface, and one from the crop region.
    b. Include the red channel of each combined render.
    c. Crop and resize the resulting tensor to 192x192 pixels [S1, unofficial].
4.  **Verification:** Write a one-byte watermark into the blue channel of the image center pixel. Read the byte back and compare against the expected value. If the frame is invalid, discard it [S1, unofficial].
5.  **Prediction:** Feed the 192x192 tensor into the MobileNetV3-Small ONNX model.
6.  **Application:** Apply the predicted `peak` value to the transfer function [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import onnxruntime_web as ort

def predict_peak(renderer, volume_data):
    # 1. Initialize ONNX Runtime
    try:
        session = ort.InferenceSession("model.onnx")
    except Exception as e:
        raise RuntimeError("ONNX Runtime Web failed to load from CDN") from e

    # 2. Sample multiple peaks
    samples = []
    peak_range = [0.1, 0.3, 0.5, 0.7, 0.9] # Example range
    for peak in peak_range:
        # Apply transfer function with current peak
        renderer.set_transfer_function_peak(peak)
        
        # Render volume
        renderer.render()
        
        # Capture pixel buffer
        color_buffer = renderer.gl.readPixels()
        
        # Capture surface-only frame
        renderer.render_surface_only()
        surface_buffer = renderer.gl.readPixels()
        
        # Verification: Check watermark
        if not verify_watermark(color_buffer):
            continue
            
        samples.append({
            'peak': peak,
            'color': color_buffer,
            'surface': surface_buffer
        })

    if not samples:
        return 0.5 # Default fallback

    # 3. Build Tensor
    # Create binary masks
    surface_mask = create_dilation_mask(samples[0]['surface'], color='green')
    crop_mask = create_crop_mask(renderer.viewport)
    
    # Combine channels: [surface_mask, crop_mask, red_channel_render_1, ..., red_channel_render_n]
    # Resize to 192x192
    tensor = build_tensor(surface_mask, crop_mask, samples)
    tensor = resize(tensor, 192, 192)

    # 4. Predict
    input_name = session.get_inputs()[0].name
    outputs = session.run(None, {input_name: tensor})
    predicted_peak = outputs[0][0][0]

    return predicted_peak

def verify_watermark(buffer):
    # Read blue channel of center pixel
    center_x = buffer.width // 2
    center_y = buffer.height // 2
    # Assuming RGBA format, blue is index 2
    watermark_byte = buffer.data[center_y * buffer.stride + center_x * 4 + 2]
    expected = 0xAA # Example expected value
    return watermark_byte == expected
```

**Parameters:**
*   **Model:** MobileNetV3-Small [S1, unofficial].
*   **Input Tensor Size:** 192x192 [S1, unofficial].
*   **ONNX Runtime Web Version:** ^1.24.3 [S1, unofficial].
*   **Transfer Function Gamma:** `dataGamma` typically 0.5; `finalGamma` typically 4.5 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Dependency Failure:** If `onnxruntime-web` is not installed, the build fails with a Webpack error [S1, unofficial].
*   **Network Timeout:** If the CDN hosting the WASM code does not respond within 10 seconds, an exception is thrown [S1, unofficial].
*   **Invalid Frames:** Frames failing the watermark verification must be discarded to prevent incorrect predictions [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dependent on the number of sampled peaks and the inference time of the MobileNetV3-Small model. Each sample requires a full render and `gl.readPixels` operation, which is expensive on the CPU-GPU boundary.
*   **Memory Complexity:** Requires storing multiple rendered frames and binary masks before tensor construction.
*   **Strengths:** Eliminates manual tuning for large datasets; works in the browser via WebAssembly.
*   **Limitations:** Requires a pre-trained model specific to the data domain (fluorescence microscopy); sensitive to network conditions for model loading; `gl.readPixels` can be a bottleneck.
*   **When to use:** When browsing many similar volume/surface pairs where manual tuning is tedious [S1, unofficial].
*   **When not to use:** When real-time interactive performance is critical and the overhead of multiple renders and inference is unacceptable.

## Background
This method is implemented in the `web-vol-viewer` project by JaneliaSciComp [S1, unofficial]. It is designed for fluorescence microscopy data visualization, building on the transfer function principles described by Wan et al. in "FluoRender" [S1, unofficial]. The approach leverages the `usePeakPredictor` React hook to automate peak selection using ONNX Runtime Web.

## Sources
[S1] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
