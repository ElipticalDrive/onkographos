# Transform Feedback GPGPU

Also called: GPGPU paradigm using transform feedback, Stream-out GPGPU

## Overview
Transform Feedback GPGPU is a general-purpose computing on graphics processing units (GPGPU) paradigm that maps algorithm inputs, processing, and outputs to the graphics pipeline by utilizing vertex and geometry shaders while completely discarding fragment shaders [S1]. It differs from common render-to-texture approaches by using transform feedback (stream-out) to record processed data directly into buffer objects, offering greater flexibility for multiple outputs compared to Multiple Render Targets (MRT) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A one-dimensional Texture Buffer Object (TBO) containing the data to be processed, and a proxy geometry stored in a Vertex Buffer Object (VBO) [S1].
*   **Output:** One or more Buffer Objects populated with processed data via transform feedback [S1].

**Data Structures:**
1.  **Input TBO:** A 1D texture buffer representing the input data in an array-like form. The size corresponds to the number of data elements to process [S1].
2.  **Proxy VBO:** A CPU-created vertex buffer containing vertices whose positions encode indices into the Input TBO. The number of vertices in the VBO equals the number of elements in the TBO [S1].
3.  **Output Buffer Object(s):** GPU memory buffers where transform feedback records vertex attributes. These can be separate buffers for each output or a single interleaved buffer [S1].

**Numbered Steps:**
1.  **Proxy Geometry Creation:** Create a proxy geometry on the CPU side. Store it in a VBO. The position of each vertex $v_i$ in the VBO encodes the index $i$ for the corresponding element in the input TBO [S1].
2.  **Data Fetching:** Bind the Input TBO. In the vertex shader, for each vertex of the proxy geometry, perform a texel fetch using the vertex position (index) to retrieve the data value from the TBO [S1].
3.  **Processing:** Render the vertices of the proxy geometry. This triggers active vertex or geometry shader threads, which serve as computation kernels. The GPU applies an identical set of operations to each input value in parallel (Single Program, Multiple Data) [S1].
    *   *Single Output:* If a single output value is produced per input, the vertex shader processes the point primitive representing the input.
    *   *Multiple Outputs:* If multiple output values are required, a geometry shader is employed as the processing kernel. The geometry shader operates on geometry primitives, allowing the creation and destruction of vertices. A point primitive from the proxy geometry (encoding a single input) is transformed into a line-strip primitive, where the vertices of the line-strip encode the array of output values [S1].
4.  **Output Recording (Transform Feedback):** Enable transform feedback (stream-out). The graphics hardware records vertex attributes for each processed primitive into the specified Output Buffer Object(s). This effectively scatters data from the vertex or geometry shading stage back to memory [S1].
5.  **Iteration:** For iterative algorithms, the Output Buffer Object containing the results of one stage is bound as the Input TBO for the subsequent stage. The CPU must manage the switching of the correct input buffer for each computation step [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
input_tbo = create_1d_texture_buffer(data_array)
proxy_vbo = create_vbo_from_indices(len(data_array))
output_buffers = [create_buffer_object() for _ in range(num_outputs)]

# Shader Program Setup
# Vertex Shader:
#   in vec3 vertex_position; # Encodes index
#   uniform samplerBuffer input_tbo;
#   out vec3 output_data;
#   void main() {
#       int index = int(vertex_position.x);
#       output_data = process(texelFetch(input_tbo, index).rgb);
#   }
# Geometry Shader (if multiple outputs):
#   in vec3 output_data[];
#   out vec3 final_output;
#   void main() {
#       for each output value:
#           final_output = output_value;
#           EmitVertex();
#   }

# Execution Loop
for stage in num_stages:
    bind_transform_feedback(output_buffers, stage)
    bind_texture_buffer(input_tbo, stage)
    bind_vertex_array(proxy_vbo)
    draw_arrays(POINTS, 0, proxy_vbo.count) # Triggers vertex/geometry shaders
    unbind_transform_feedback()
    
    # Prepare for next stage
    input_tbo = output_buffers[stage]
```

**Parameters and Typical Values:**
*   **Proxy Geometry Type:** Points (for single output) or Lines (for multiple outputs via geometry shader) [S1].
*   **Buffer Interleaving:** Output values can be recorded to separate buffers or interleaved into a single buffer. The choice depends on hardware architecture performance characteristics [S1].

**Edge Cases and Pitfalls:**
*   **Hardware Performance Variance:** The performance of recording output values to separate buffers versus interleaving them into a single buffer varies across different hardware architectures [S1].
*   **Shader Unification:** This approach is only efficient when shader processing is unified (e.g., Unified Shader Model or Shader Model 4.0+), as it relies heavily on vertex and geometry shaders [S1].
*   **CPU-Side Management:** The correct input buffer for each computation stage in an iterative approach must be determined and managed by the CPU [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Performance varies based on hardware architecture, specifically regarding the choice between separate and interleaved output buffers [S1]. Efficiency depends on unified shader processing capabilities [S1].
*   **Strengths:** Intuitive mapping of algorithm steps (Input/Processing/Output) to GPU pipeline stages; flexible output handling (separate or interleaved buffers) compared to MRT; supports iterative approaches easily [S1].
*   **Limitations:** Discards fragment shaders, which may be inefficient on older or non-unified hardware architectures; requires careful CPU-side management for iterative buffer switching [S1].
*   **When to use:** When implementing general computations on the GPU that require flexible output structures (multiple outputs) and benefit from the simplicity of vertex/geometry shader processing, particularly in frameworks like VolFliesGPU for illustrative volume rendering [S1].
*   **When not to use:** On hardware without unified shader models where vertex/geometry shader performance is significantly lower than fragment shader performance; when fragment-based processing is inherently more suitable for the algorithm [S1].

## Background
The Transform Feedback GPGPU paradigm was proposed by R.F.P. van Pelt, A. Vilanova, and H.M.M. van Wijk in 2008 as part of the VolFliesGPU framework for interactive illustrative volume rendering [S1]. It was introduced to provide a more intuitive and flexible alternative to common render-to-texture approaches, leveraging new extensions in the shader model (transform feedback/stream-out) to map general algorithms to the stream processing pipeline [S1].

## Sources
[S1] GPU-based Particle Systems for Illustrative Volume Rendering, https://doi.org/10.2312/vg/vg-pbg08/089-096, fetched 2026-10-26
