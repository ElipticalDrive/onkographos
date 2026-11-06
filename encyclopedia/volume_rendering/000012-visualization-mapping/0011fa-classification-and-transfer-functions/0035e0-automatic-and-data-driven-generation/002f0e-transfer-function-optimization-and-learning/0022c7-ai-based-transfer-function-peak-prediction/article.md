# AI-Based Transfer Function Peak Prediction

Also called: AI-Assisted Transfer Function Selection, ONNX Transfer Function Picker, Peak Predictor, usePeakPredictor, usePeakPredictor hook

## Overview
This method automates the selection of the `peak` parameter for a volume rendering transfer function by analyzing rendered frames using a lightweight neural network [S1, unofficial]. It addresses the tedious manual tuning required when browsing multiple volume/surface pairs, particularly for fluorescence microscopy data where the `peak` controls the opacity rise in a "tent" shape [S1, unofficial]. Unlike optimization-based methods that require differentiable rendering and gradient backpropagation, this prediction-based approach uses a pre-trained MobileNetV3-Small model running in WebAssembly to infer the optimal parameter from pixel buffers [S1, unofficial].

## How it works
The method operates as a React hook (`usePeakPredictor`) that intercepts the rendering pipeline to capture frames, process them into a tensor, and query an ONNX model for the optimal `peak` value.

**Inputs and Outputs**
*   **Input:** Rendered pixel buffers (combined volume/surface and surface-only) from the WebGL context.
*   **Output:** A scalar `peak` parameter (integer or float, typically 0–255 for 8-bit data) applied to the transfer function.

**Data Structures**
1.  **Transfer Function Texture:** Implemented as a `THREE.DataTexture` with width 256, height 1, and format `THREE.RGBAFormat`. Each 8-bit data value maps to a 4-byte RGBA entry [S1, unofficial].
2.  **Transfer Function Shape:** The alpha channel follows a tent shape:
    *   Rises from `alpha0` (at data value 0) to 255 (at data value `peak`).
    *   Falls to `alpha1` (at data value 255).
    *   The rise/fall curve is controlled by `dataGamma` using $y^{(1.0 / \text{dataGamma})}$, where $y$ is the normalized data value [S1, unofficial].
3.  **Input Tensor:** A multi-channel tensor created from captured frames, resized to $192 \times 192$ pixels.
4.  **ONNX Model:** MobileNetV3-Small, exported to ONNX format, running via ONNX Runtime Web [S1, unofficial].

**Numbered Steps**
1.  **Initialization:** The hook initializes the ONNX Runtime Web environment. It fetches the WASM code from a CDN; if the CDN fails to respond within 10 seconds, an exception is thrown [S1, unofficial].
2.  **Capture Loop:** The hook cycles through a range of `peak` parameters (high to low) to generate reference renders.
3.  **Frame Capture:** For each `peak` candidate, the renderer captures the pixel buffer using `gl.readPixels`.
    *   **Verification:** A fragment shader writes a one-byte watermark into the blue channel of the image center pixel. The predictor reads this byte back and compares it against an expected value to verify frame integrity before processing [S1, unofficial].
4.  **Tensor Construction:**
    *   Capture a "surface-only" frame to help the model locate the green surface.
    *   Create two binary masks: one from the dilation of the green surface, and one for the crop region.
    *   Extract the red channel from each combined render (volume + surface).
    *   Combine these channels into a single tensor.
    *   Crop and resize the tensor to $192 \times 192$ pixels [S1, unofficial].
5.  **Prediction:** Feed the tensor into the MobileNetV3-Small ONNX model.
6.  **Application:** The model returns the predicted `peak` value. The hook applies this value to the transfer function texture [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the usePeakPredictor hook logic
# Note: This is a simplified representation of the React hook's internal cycle.

def use_peak_predictor(renderer, gl, model_path):
    # 1. Initialize ONNX Runtime
    try:
        session = onnxruntime.InferenceSession(model_path, providers=['WebAssemblyExecutionProvider'])
    except Exception as e:
        raise RuntimeError("ONNX Runtime Web failed to initialize. Check CDN availability.") from e

    # 2. Define Transfer Function Parameters
    # The TF is a tent shape defined by alpha0, peak, dataGamma, alpha1
    alpha0 = 0
    alpha1 = 255
    data_gamma = 0.5
    final_gamma = 4.5
    color_str = "#ff00ff"

    # 3. Capture and Process Frames
    # The hook cycles through peak values to find the best one, 
    # but the core prediction step uses a single representative frame 
    # or a sequence to infer the optimal peak. 
    # Based on notes, it captures frames for *each* TF candidate to build the tensor?
    # Re-reading: "Cycles through several transfer functions... Captures resulting pixel buffers... Feeds tensor to ONNX model."
    # This implies the input to the model might be a composite or the model predicts based on the current view.
    # However, standard "Peak Prediction" implies predicting the *best* peak for the *current* data.
    # The notes say: "Cycles through several transfer functions... Captures... Builds a multi-channel tensor... Feeds tensor... Model returns predicted peak value."
    # This suggests the input tensor contains information from multiple peaks or the model is trained to output the peak given the scene.
    # Let's assume the tensor is built from the *current* render state or a specific capture strategy.
    # Actually, the note "Cycles through several transfer functions" might refer to the *training* or *calibration* phase, 
    # OR the hook tries multiple peaks to generate a tensor that represents the "best" view?
    # Let's stick strictly to the text: "Cycles through several transfer functions... Captures... Builds tensor... Feeds tensor... Model returns predicted peak."
    # This implies the input tensor is derived from multiple captures? Or is it a single capture?
    # "Builds a multi-channel tensor from captures: two binary masks... plus red channel of each combined render."
    # "Each combined render" implies multiple renders.
    # So the input tensor is a stack of frames?
    
    # Let's assume the hook captures N frames with different peaks to create a rich input tensor.
    num_peaks = 5 # Example: high to low
    peak_values = [255, 200, 150, 100, 50]
    
    tensor_channels = []
    
    for i, peak in enumerate(peak_values):
        # Set TF
        tf_texture = make_fluo_transfer_tex(alpha0, peak, data_gamma, alpha1, color_str)
        renderer.set_transfer_function(tf_texture)
        
        # Render and Capture
        renderer.render()
        buffer = gl.readPixels()
        
        # Verification
        watermark = buffer[center_pixel_blue_channel]
        if watermark != expected_watermark:
            continue # Skip invalid frame
            
        # Extract Red Channel
        red_channel = extract_red_channel(buffer)
        tensor_channels.append(red_channel)
        
    # Surface-only capture for masks
    renderer.render_surface_only()
    surface_buffer = gl.readPixels()
    green_mask = create_binary_mask(surface_buffer, color="green")
    crop_mask = create_crop_mask()
    
    # Dilate green mask
    green_mask_dilated = dilate(green_mask)
    
    # Combine into tensor
    # Channels: green_mask_dilated, crop_mask, red_channel_1, ..., red_channel_N
    input_tensor = stack_channels([green_mask_dilated, crop_mask] + tensor_channels)
    
    # Resize to 192x192
    input_tensor = resize(input_tensor, 192, 192)
    
    # Predict
    predicted_peak = session.run(None, {session.get_inputs()[0].name: input_tensor})[0]
    
    # Apply
    final_tf = make_fluo_transfer_tex(alpha0, predicted_peak, data_gamma, alpha1, color_str)
    renderer.set_transfer_function(final_tf)
```

**Parameters**
*   **Model:** MobileNetV3-Small [S1, unofficial].
*   **Input Tensor Size:** $192 \times 192$ pixels [S1, unofficial].
*   **ONNX Runtime Web Version:** ^1.24.3 [S1, unofficial].
*   **Transfer Function Gamma:** `dataGamma` typically 0.5; `finalGamma` typically 4.5 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Dependency Failure:** ONNX Runtime Web is an optional peer dependency. If omitted, the build fails with a Webpack error: `Module not found: Error: Can't resolve 'onnxruntime-web'...` [S1, unofficial].
*   **Network Timeout:** The WASM code is fetched from a CDN. If the CDN does not respond within 10 seconds, an exception is thrown [S1, unofficial].
*   **Precision:** The method relies on 8-bit data values mapped to a 256-width texture. Precision loss may occur if the data range is not properly normalized before mapping to the texture [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dependent on the number of frames captured and the inference time of MobileNetV3-Small on WebAssembly. Capturing multiple frames and resizing adds overhead.
*   **Memory Complexity:** Low, as the model is small (MobileNetV3-Small) and tensors are resized to $192 \times 192$.
*   **Strengths:** Eliminates manual tuning for fluorescence microscopy data; runs in the browser via WebAssembly; lightweight compared to full optimization-based methods.
*   **Limitations:** Requires a specific data distribution (fluorescence microscopy); dependent on CDN availability for WASM; requires `gl.readPixels` which can be slow on some hardware; limited to predicting a single parameter (`peak`) rather than the full TF.
*   **When to use:** When browsing many volume/surface pairs in a web-based viewer and manual tuning is prohibitive.
*   **When not to use:** For non-fluorescence data where the "tent" shape is not appropriate; when network connectivity is unreliable.

## Background
The method is implemented in the `web-vol-viewer` project by JaneliaSciComp [S1, unofficial]. It is based on the transfer function design described by Wan et al. in "FluoRender: An Application of 2D Image Space Methods for 3D and 4D Confocal Microscopy Data Visualization in Neurobiology Research" (2012 IEEE Pacific Visualization Symposium) [S1, unofficial]. The desktop application VVD Viewer also uses this transfer function [S1, unofficial]. The AI-based prediction layer was added to automate the selection of the `peak` parameter for the web viewer.

## Sources
[S1] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
