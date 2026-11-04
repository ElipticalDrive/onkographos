# Sigma Network Layer


## Overview
The Sigma Network Layer is the final component of the neural volume rendering pipeline, responsible for predicting the volume density ($\sigma$) at a given 3D position. It builds on the [RGB Network Layer](4051-rgb-network-layer.md) by processing the same positional embeddings but producing a scalar density value rather than color. It is distinct from the RGB layer in its output dimensionality and activation function, serving as the basis for opacity accumulation in ray marching.

## How it works
**Inputs and Outputs:**
- **Input:** A batch of 3D positions ($xyz$) and their corresponding positional embeddings. The embeddings are pre-computed by the parent network layer.
- **Output:** A scalar density value ($\sigma$) for each input position.

**Data Structures:**
The implementation utilizes a Taichi kernel structure with specific shared memory buffers to optimize performance on GPU hardware.
- `self.xyzs_embedding`: A 2D array storing the positional embeddings for all rays. Layout: `[num_rays, sigma_n_input]`.
- `self.sigma_weights`: A 1D array containing the flattened weights of the MLP.
- `weight`: A shared memory array of size `sigma_sm_preload` used to cache weights for the current block.
- `input_val`: A shared memory array of size `[sigma_n_input, block_dim]` to hold input embeddings for the current thread block.
- `hid1`: A shared memory array of size `[net_width, block_dim]` storing the output of the first hidden layer.
- `hid2`: A shared memory array of size `[sigma_n_output, block_dim]` storing the output of the second hidden layer.
- `self.out_1`: An output array indexed by `self.temp_hit[sn]`, storing the final density.
- `self.final_embedding`: An array storing intermediate hidden layer outputs, layout `[num_rays, sigma_n_output]`.

**Parameters:**
- `block_dim`: The number of threads per block.
- `self.sigma_n_input`: The dimensionality of the input embeddings.
- `self.sigma_model_size`: Total size of the model weights.
- `self.net_width`: The width of the hidden layers.
- `self.sigma_sm_preload`: The number of weights preloaded into shared memory.
- `self.sigma_n_output`: The number of output neurons (typically 1 for density).
- `self.sigma_layer1_base`: The offset in the weight array where the second layer's weights begin.
- `data_type`: The floating-point type used for computation (e.g., `float32`).

**Algorithm:**
The layer processes rays in blocks. For each block index `sn`:

1.  **Thread Identification:** Calculate the thread ID within the block: `tid = sn % block_dim`.
2.  **Weight Preloading:** Preload the necessary network weights into the shared memory array `weight`. Iterate `k` from `0` to `self.sigma_sm_preload - 1`:
    $$ weight[k] = self.sigma\_weights[k] $$
3.  **Input Loading:** Load the positional embeddings for the current block into shared memory. Iterate `i` from `0` to `self.sigma_n_input - 1$:
    $$ input\_val[i, tid] = self.xyzs\_embedding[sn, i] $$
4.  **Synchronization:** Synchronize all threads in the block to ensure data is loaded:
    $$ ti.simt.block.sync() $$
5.  **First Hidden Layer:** If `sn < did_launch_num`, compute the first hidden layer. For each neuron `i` in `range(self.net_width)`:
    $$ temp = 0 $$
    For `j` in `range(self.sigma_n_input)`:
    $$ temp += input\_val[j, tid] \times weight[i \times self.sigma\_n\_input + j] $$
    Store the result:
    $$ hid1[i, tid] = temp $$
6.  **Second Hidden Layer (Output):** Compute the output layer. For each output `i` in `range(self.sigma_n_output)`:
    $$ temp = 0 $$
    For `j` in `range(self.net_width)`:
    $$ temp += data\_type(\max(0.0, hid1[j, tid])) \times weight[self.sigma\_layer1\_base + i \times self.net\_width + j] $$
    Store the result:
    $$ hid2[i, tid] = temp $$
7.  **Final Density and Storage:** Compute the final density using the exponential activation function and store it.
    $$ self.out\_1[self.temp\_hit[sn]] = data\_type(\exp(hid2[0, tid])) $$
    Store all hidden layer outputs for potential downstream use:
    $$ self.final\_embedding[sn, i] = hid2[i, tid] $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
@ti.kernel
def sigma_network_layer():
    for sn in ti.ndrange(self.N_rays):
        tid = sn % block_dim
        
        # Preload weights
        for k in ti.static(range(self.sigma_sm_preload)):
            weight[k] = self.sigma_weights[k]
            
        # Load inputs
        for i in ti.static(range(self.sigma_n_input)):
            input_val[i, tid] = self.xyzs_embedding[sn, i]
            
        ti.simt.block.sync()
        
        if sn < did_launch_num:
            # First hidden layer
            for i in ti.static(range(self.net_width)):
                temp = 0.0
                for j in ti.static(range(self.sigma_n_input)):
                    temp += input_val[j, tid] * weight[i * self.sigma_n_input + j]
                hid1[i, tid] = temp
                
            # Second hidden layer
            for i in ti.static(range(self.sigma_n_output)):
                temp = 0.0
                for j in ti.static(range(self.net_width)):
                    # ReLU activation
                    relu_val = max(0.0, hid1[j, tid])
                    temp += data_type(relu_val) * weight[self.sigma_layer1_base + i * self.net_width + j]
                hid2[i, tid] = temp
                
            # Final output
            self.out_1[self.temp_hit[sn]] = data_type(exp(hid2[0, tid]))
            
            # Store embeddings
            for i in ti.static(range(self.sigma_n_output)):
                self.final_embedding[sn, i] = hid2[i, tid]
```

**Edge Cases and Pitfalls:**
- **Precision:** The use of `data_type` suggests potential mixed-precision or specific hardware optimizations. Ensure `exp` and `max` operations are consistent with the chosen data type to avoid NaNs or infinities for extreme density values.
- **Boundary Checks:** The condition `sn < did_launch_num` handles cases where the number of rays is not a multiple of the block dimension or where some rays are invalid.
- **Shared Memory Limits:** `sigma_sm_preload` must be chosen carefully to fit within the GPU's shared memory limits, considering the block dimension.

## Operating characteristics
- **Time Complexity:** $O(N \cdot (D_{in} \cdot W + W \cdot D_{out}))$ where $N$ is the number of rays, $D_{in}$ is input embedding dimension, $W$ is network width, and $D_{out}$ is output dimension. The shared memory optimization reduces global memory bandwidth pressure.
- **Memory Complexity:** $O(W \cdot D_{in} + W \cdot D_{out})$ for weights, plus $O(D_{in} \cdot B + W \cdot B + D_{out} \cdot B)$ for shared memory buffers per block, where $B$ is block dimension.
- **Performance:** Optimized for GPU execution using Taichi's shared memory primitives. Performance scales with block dimension and shared memory availability.
- **Strengths:** High throughput for density prediction, efficient use of shared memory.
- **Limitations:** Requires careful tuning of shared memory sizes. Dependent on the efficiency of the parent embedding layer.

## Background
The Sigma Network Layer is part of the neural volume rendering pipeline, specifically within the context of Neural Graphics Primitives (NGP) and their implementations. The specific Taichi-based implementation described here is from an unofficial repository [S1, unofficial]. The concept of using MLPs for density prediction in volume rendering has been prevalent in neural radiance fields (NeRF) and its variants.

## Sources
[S1] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
