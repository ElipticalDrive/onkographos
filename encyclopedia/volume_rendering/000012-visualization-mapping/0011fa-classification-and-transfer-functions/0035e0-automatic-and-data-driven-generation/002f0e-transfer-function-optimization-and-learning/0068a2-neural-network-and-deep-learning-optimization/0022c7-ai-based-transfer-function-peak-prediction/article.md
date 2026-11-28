# AI-Based Transfer Function Peak Prediction

Also called: AI-Assisted Transfer Function Selection, ONNX Transfer Function Picker, Peak Predictor, usePeakPredictor, usePeakPredictor hook

## Overview
This method automates the selection of the `peak` parameter for a volume rendering transfer function by analyzing rendered frames using a neural network [S1, unofficial]. It solves the problem of tedious manual parameter tuning when browsing many volume/surface pairs [S1, unofficial]. It operates in image space using a pre-trained ONNX model running in WebAssembly [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Rendered frames from multiple transfer function configurations with varying `peak` values, plus a surface-only frame [S1, unofficial].
*   **Output:** A scalar value representing the predicted optimal `peak` parameter [S1, unofficial].

**Data Structures:**
The transfer function is implemented as a `DataTexture` with width 256, height 1, and `RGBAFormat` (4 bytes per 8-bit data value) [S1, unofficial]. The alpha channel follows a tent shape: it rises from `alpha0` at data value 0 to 255 at `peak`, then falls to `alpha1` at data value 255 [S1, unofficial]. The rise and fall shape is controlled by `dataGamma` via the power function $y^{(1.0 / \text{dataGamma})}$, where $y$ is the data value normalized to $[0, 1]$ [S1, unofficial]. After ray casting, a `finalGamma` is applied to the accumulated color $c$ via $c^{(1.0 / \text{finalGamma})}$ [S1, unofficial].

**Steps:**
1.  **Initialization:** Load the ONNX Runtime Web environment [S1, unofficial].
2.  **Sampling:** Cycle through a range of `peak` values from high to low [S1, unofficial].
3.  **Rendering & Capture:** For each `peak`, apply the transfer function, render the volume, and capture the pixel buffer using `gl.readPixels` [S1, unofficial]. Also capture a surface-only frame to locate the green surface [S1, unofficial].
4.  **Verification:** Read the blue channel of the image center pixel to check for a one-byte watermark written by the fragment shader. Compare against the expected value; discard invalid frames [S1, unofficial].
5.  **Tensor Construction:** Build a multi-channel tensor from the valid captures: two binary masks (dilation of the green surface, crop region) plus the red channel of each combined render. Crop and resize the tensor to 192x192 pixels [S1, unofficial].
6.  **Prediction:** Feed the 192x192 tensor into the MobileNetV3-Small ONNX model [S1, unofficial].
7.  **Application:** Apply the predicted `peak` value returned by the model to the transfer function [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def resize_tensor(tensor, target_h, target_w):
    # [G] Standard bilinear resize implementation
    h, w, c = tensor.shape
    src_h, src_w = float(h), float(w)
    dst_h, dst_w = float(target_h), float(target_w)
    resized = np.zeros((target_h, target_w, c), dtype=tensor.dtype)
    for y in range(target_h):
        for x in range(target_w):
            src_y = (y + 0.5) * (src_h / dst_h) - 0.5
            src_x = (x + 0.5) * (src_w / dst_w) - 0.5
            y0, x0 = int(np.floor(src_y)), int(np.floor(src_x))
            y1, x1 = min(y0 + 1, h - 1), min(x0 + 1, w - 1)
            dy, dx = src_y - y0, src_x - x0
            for ch in range(c):
                resized[y, x, ch] = (1 - dy) * (1 - dx) * tensor[y0, x0, ch] + \
                                    (1 - dy) * dx * tensor[y0, x1, ch] + \
                                    dy * (1 - dx) * tensor[y1, x0, ch] + \
                                    dy * dx * tensor[y1, x1, ch]
    return resized

def predict_peak(renderer, expected_watermark):
    # 1. Initialize ONNX Runtime Web
    session = ort.InferenceSession("model.onnx")

    samples = []
    peak_range = [0.9, 0.7, 0.5, 0.3, 0.1] # High to low

    # 2 & 3. Sampling, Rendering & Capture
    for peak in peak_range:
        renderer.set_transfer_function_peak(peak)
        renderer.render()
        color_buffer = renderer.gl.readPixels()

        renderer.render_surface_only()
        surface_buffer = renderer.gl.readPixels()

        # 4. Verification
        center_x = color_buffer.shape[1] // 2
        center_y = color_buffer.shape[0] // 2
        # Blue channel is index 2 in RGBA
        watermark_byte = color_buffer[center_y, center_x, 2]
        if watermark_byte != expected_watermark:
            continue

        samples.append({
            'peak': peak,
            'color': color_buffer,
            'surface': surface_buffer
        })

    if not samples:
        return 0.5 # [G] Default fallback if no valid frames

    # 5. Tensor Construction
    # Create binary masks
    surface_mask = np.zeros_like(surface_buffer[:, :, 0], dtype=np.uint8)
    surface_mask[surface_buffer[:, :, 1] > 0] = 1 # Green channel > 0

    crop_mask = np.ones_like(surface_mask, dtype=np.uint8) # [G] Full crop region mask

    # Combine channels: [surface_mask, crop_mask, red_channel_render_1, ..., red_channel_render_n]
    channels = [surface_mask, crop_mask]
    for sample in samples:
        channels.append(sample['color'][:, :, 0]) # Red channel
    tensor = np.stack(channels, axis=-1)

    # Crop and resize to 192x192
    tensor = tensor[:192, :192, :]
    tensor = resize_tensor(tensor, 192, 192)

    # 6. Prediction
    input_name = session.get_inputs()[0].name
    outputs = session.run(None, {input_name: tensor})
    predicted_peak = outputs[0][0][0]

    return predicted_peak
```

**Parameters:**
*   **Model:** MobileNetV3-Small [S1, unofficial].
*   **Input Tensor Size:** 192x192 [S1, unofficial].
*   **ONNX Runtime Web Version:** ^1.24.3 [S1, unofficial].
*   **Transfer Function Gamma:** `dataGamma` typically 0.5; `finalGamma` typically 4.5 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Dependency Failure:** If `onnxruntime-web` is omitted, the build fails with a Webpack error [S1, unofficial].
*   **Network Timeout:** WASM code is fetched from a CDN; if the CDN does not respond within 10 seconds, an exception is thrown [S1, unofficial].
*   **Manual Tuning:** Manual selection of the `peak` parameter is tedious when browsing many volume/surface pairs [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Each sample requires a full render and `gl.readPixels` operation, which is expensive on the CPU-GPU boundary [S1, unofficial].
*   **Memory Complexity:** Not given in the sources. Requires storing multiple rendered frames and binary masks before tensor construction [S1, unofficial].
*   **Strengths:** Eliminates manual tuning for large datasets; works in the browser via WebAssembly [S1, unofficial].
*   **Limitations:** Requires a pre-trained model specific to the data domain (fluorescence microscopy); sensitive to network conditions for model loading; `gl.readPixels` can be a bottleneck [S1, unofficial].
*   **When to use:** When browsing many similar volume/surface pairs where manual tuning is tedious [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
This method is implemented in the `web-vol-viewer` project by JaneliaSciComp [S1, unofficial]. It is designed for fluorescence microscopy data visualization, building on the transfer function principles described by Wan et al. in "FluoRender" [S1, unofficial]. The approach leverages the `usePeakPredictor` React hook to automate peak selection using ONNX Runtime Web [S1, unofficial].

## Sources
[S1] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
