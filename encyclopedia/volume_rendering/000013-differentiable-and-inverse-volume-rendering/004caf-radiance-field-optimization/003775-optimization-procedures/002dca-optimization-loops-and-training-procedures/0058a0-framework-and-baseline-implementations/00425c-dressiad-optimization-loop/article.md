# DressiAD Optimization Loop

Also called: Dressi-AD Computational Graph Execution, DressiAD Training Loop, Optimization Loop

## Overview
The DressiAD Optimization Loop is a define-and-run differentiable rendering optimization framework that constructs a computational graph, marks variables for gradient tracking, and executes forward/backward passes using Stochastic Gradient Descent (SGD) [S1]. It distinguishes itself from standard AD libraries by packing the computational graph into Vulkan command buffers for execution, allowing for hardware-agnostic differentiable rendering [S1]. Unlike general-purpose AD which may suffer from high memory usage due to storing intermediate states, DressiAD manages graph staticity and rebuilds stages only when necessary, optimizing for the specific constraints of GPU-based rendering [S1].

## How it works
The method operates by defining a computational graph of rendering and loss functions, then executing an iterative optimization loop that handles data transfer between CPU and GPU, graph traversal for gradient computation, and Vulkan command submission [S1].

### Inputs and Outputs
**Inputs:**
*   Scene data: Vertex positions, UVs, faces, model/view/projection matrices, environment map, albedo, metallic, roughness, normal maps, and background [S1].
*   Target image: The ground-truth image to minimize loss against [S1].
*   HardSoftRas parameters: $K$ (number of peeling steps), $r$ (radius), $\sigma$ (blending parameter), $\delta$ (silhouette edge width) [S1].
*   Learning rate: $lr$ [S1].

**Outputs:**
*   Optimized scene parameters (e.g., vertex positions, normal maps) saved as a GLTF file [S1].

### Data Structures
**CpuImage:**
A simple buffer representing a 2D image used to store scene data on the CPU [S1]. It holds vertex positions, UVs, faces, matrices, environment maps, albedo, metallic, roughness, normal, and background data [S1].

**Variable:**
A data structure representing inputs and outputs of functions in the computational graph [S1].
*   **Fields:**
    *   `type`: Enum indicating data type (`VEC3`, `VEC2`, `IVEC3`, `MAT4`, `FLOAT`, `VEC4`) [S1].
    *   `size`: Dimensions of the variable [S1].
    *   `requires_grad`: Boolean flag indicating if the variable is an optimization target [S1].
    *   `dirty`: Boolean flag indicating if the variable's data has changed since the last GPU sync [S1].

**DressiAD Class:**
Manages the optimization state.
*   **Fields:**
    *   `m_loss_var`: The loss variable [S1].
    *   `m_init_status`: Enum tracking initialization phase (`BACKWARD`, `STAGE`, `SUBSTAGE`, `FINISHED`) [S1].
    *   `m_graph_static_cnt`: Counter for static graph iterations [S1].
    *   `m_vk_cmd_buf`: Vulkan command buffer [S1].
    *   `m_input_vars`: List of input variables [S1].
    *   `m_input_grad_vars`: List of gradient variables for inputs [S1].

### Parameters
*   $lr$: Learning rate for SGD. Typical value: $0.01$ [S1].
*   $K$: Number of peeling steps for HardSoftRas. Typical value: $2$ [S1].
*   $r$: Radius parameter for HardSoftRas. Typical value: $0.01$ [S1].
*   $\sigma$: Blending parameter for HardSoftRas. Calculated as $r / 7$ [S1].
*   $\delta$: Silhouette edge width for HardSoftRas. Equal to $r$ [S1].

### Algorithm Steps
1.  **Initialization:** Create a `DressiBasicRenderer` instance and load scene data from files (e.g., `.gltf`, `.png`) into CPU memory as `CpuImage` buffers [S1].
2.  **Graph Construction:** Call `buildGraph` with scene variables and HardSoftRas parameters ($K, r, \sigma, \delta$). This calls `BuildBasicRenderGraph` to assemble operators and `F::` functions into a computational graph [S1].
3.  **Loss Definition:** Compute the L1 loss between the target image and the rendered image:
    $$ L = \text{Mean}(|I_{\text{target}} - I_{\text{rendered}}|) $$
    Set this loss variable in the `DressiAD` instance [S1].
4.  **Optimizer Definition:** Define an SGD optimizer function that updates variables $x$ using gradients $g_x$:
    $$ x_{\text{new}} = x - g_x \cdot lr $$
    Set this optimizer in the `DressiAD` instance [S1].
5.  **Target Marking:** Mark specific variables (e.g., vertex positions, normal maps) as optimization targets by setting their `requires_grad` flag [S1].
6.  **Optimization Loop:** For each iteration:
    a.  **Dirty Check:** Check if any input variable has changed dirty flags. If so, reset `m_graph_static_cnt` to 0 [S1].
    b.  **Rebuild Check:**
        *   If `m_graph_static_cnt` equals `FAST_REBUILD_COUNT`, set `m_init_status` to `STAGE` [S1].
        *   If `m_graph_static_cnt` equals `FULL_REBUILD_COUNT`, set `m_init_status` to `SUBSTAGE` [S1].
    c.  **Data Transfer:** If `m_is_sent` is false, send CPU images to GPU using `sendImg`, which marks variables as dirty and transfers data [S1].
    d.  **Execution:** Call `execStep()` [S1].
    e.  **Backward Pass Construction:** Inside `execStep`, if `m_init_status <= BACKWARD`:
        *   Initialize `fwd_bwds_map` (mapping forward variables to backward gradients) and `func_queue` [S1].
        *   Set the gradient of the loss variable to 1.0: `fwd_bwds_map[loss_var] = {1.0}` [S1].
        *   Push the loss variable's creator function to `func_queue` [S1].
        *   While `func_queue` is not empty:
            *   Pop function $f$ from queue [S1].
            *   Call `f.buildBwd()` to generate backward graph nodes [S1].
            *   For each input $x$ of $f$ with gradient $g_x$:
                *   Add $g_x$ to `fwd_bwds_map[x]` [S1].
                *   If $x$ has a creator, push it to `func_queue` [S1].
    f.  **Gradient Aggregation:** Sum gradients for each input variable:
        $$ g_{\text{total}} = \sum g_x $$
        Store pairs of `(input_var, total_gradient)` [S1].
    g.  **Optimizer Step:** Apply the optimizer function to update the input variables [S1].
    h.  **Vulkan Submission:** Submit the command buffer `m_vk_cmd_buf` using `VkQueueSubmit` [S1].
    i.  **Cleanup:** Mark all input variables as clean recursively (`setDirtyRecursively(false)`) and set `m_init_status` to `FINISHED` [S1].
    j.  Increment `m_graph_static_cnt` [S1].
7.  **Finalization:** Save the optimized scene data as a GLTF file [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
FAST_REBUILD_COUNT = 10  # Example value, not explicitly given in sources but implied by logic
FULL_REBUILD_COUNT = 100 # Example value, not explicitly given in sources but implied by logic

class Variable:
    def __init__(self, name, type, size):
        self.name = name
        self.type = type
        self.size = size
        self.requires_grad = False
        self.dirty = False
        self.creator = None
        self.grads = []

    def set_requires_grad(self):
        self.requires_grad = True
        # Recursively mark dependencies if needed, though sources say setRequiresGradRecursively
        pass

    def set_dirty(self, val):
        self.dirty = val

    def set_dirty_recursively(self, val):
        self.dirty = val
        if self.creator:
            self.creator.set_dirty_recursively(val)

class Function:
    def __init__(self, name, forward_func, backward_generator):
        self.name = name
        self.forward_func = forward_func
        self.backward_generator = backward_generator
        self.inputs = []
        self.outputs = []

    def build_bwd(self, output_grads):
        # Generate backward graph nodes
        # Return list of (input_var, gradient) pairs
        pass

class DressiAD:
    def __init__(self):
        self.loss_var = None
        self.optimizer_func = None
        self.init_status = "BACKWARD"
        self.graph_static_cnt = 0
        self.vk_cmd_buf = None
        self.input_vars = []
        self.input_grad_vars = []
        self.fwd_bwds_map = {}
        self.func_queue = []

    def set_loss_var(self, loss_var):
        self.loss_var = loss_var
        self.init_status = "BACKWARD"

    def set_optimizer(self, optimizer_func):
        self.optimizer_func = optimizer_func

    def exec_step(self):
        # Check dirty flags
        if any(v.dirty for v in self.input_vars):
            self.graph_static_cnt = 0

        # Check rebuild condition
        if self.graph_static_cnt == FAST_REBUILD_COUNT:
            self.init_status = "STAGE"
        elif self.graph_static_cnt == FULL_REBUILD_COUNT:
            self.init_status = "SUBSTAGE"

        # Build backward pass if needed
        if self.init_status <= "BACKWARD":
            self.fwd_bwds_map = {}
            self.func_queue = []
            
            # Set loss gradient to 1.0
            self.fwd_bwds_map[self.loss_var] = [1.0]
            
            # Push loss creator
            if self.loss_var.creator:
                self.func_queue.append(self.loss_var.creator)

            while self.func_queue:
                func = self.func_queue.pop()
                # Get gradients for this function's outputs
                output_grads = []
                for out in func.outputs:
                    if out in self.fwd_bwds_map:
                        output_grads.extend(self.fwd_bwds_map[out])
                
                # Build backward
                grad_inputs = func.build_bwd(output_grads)
                
                for inp, grad in grad_inputs:
                    if inp not in self.fwd_bwds_map:
                        self.fwd_bwds_map[inp] = []
                    self.fwd_bwds_map[inp].append(grad)
                    
                    if inp.creator:
                        self.func_queue.append(inp.creator)

        # Aggregate gradients
        input_vars = []
        input_grad_vars = []
        for var, grads in self.fwd_bwds_map.items():
            if var.requires_grad:
                input_vars.append(var)
                # Sum gradients pixel-wise
                total_grad = sum(grads)
                input_grad_vars.append(total_grad)

        # Apply optimizer
        if self.optimizer_func and input_vars:
            updated_vars = self.optimizer_func(input_vars, input_grad_vars)
            # Update variable values
            for var, new_val in zip(input_vars, updated_vars):
                var.value = new_val

        # Vulkan submission
        VkQueueSubmit(self.vk_cmd_buf)

        # Cleanup
        for var in self.input_vars:
            var.set_dirty_recursively(False)
        self.init_status = "FINISHED"
        self.graph_static_cnt += 1

class DressiBasicRenderer:
    def __init__(self):
        self.dressi_ad = DressiAD()
        self.var_map = {}
        self.img_map = {}
        self.is_sent = False

    def load_scene(self, scene_file, target_img_file):
        # Load scene data into CpuImage buffers
        # Parse GLTF/PNG
        pass

    def build_graph(self, K, r, sigma, delta, lr):
        # Build rendering computational graph
        rendered_img = BuildBasicRenderGraph(
            self.var_map["vtx_pos"], self.var_map["vtx_uv"], self.var_map["faces"],
            self.var_map["model_mat"], self.var_map["view_mat"],
            self.var_map["prj_mat"], self.var_map["env_img"],
            self.var_map["albedo"], self.var_map["metallic"],
            self.var_map["roughness"], self.var_map["normal"],
            self.var_map["background"], K, r, sigma, delta
        )
        
        # Compute L1 loss
        loss = F::Mean(F::Abs(self.var_map["target"] - rendered_img))
        
        # Set loss and optimizer
        self.dressi_ad.set_loss_var(loss)
        
        # Define SGD optimizer
        def sgd_optimizer(xs, gxs):
            updated_xs = []
            for x, gx in zip(xs, gxs):
                updated_xs.append(x - gx * lr)
            return updated_xs
        
        self.dressi_ad.set_optimizer(sgd_optimizer)

    def set_requires_grad(self, name):
        self.var_map[name].set_requires_grad()

    def exec_step(self):
        if not self.is_sent:
            for name, var in self.var_map.items():
                if name in self.img_map:
                    var.set_dirty(True)
                    SendHostImageToDevice(var, self.img_map[name])
            self.is_sent = True
        
        self.dressi_ad.exec_step()

    def save_scene(self, output_file):
        # Save optimized data as GLTF
        pass

def main():
    renderer = DressiBasicRenderer()
    renderer.load_scene("initial_scene.gltf", "target_img.png")
    
    K = 2
    r = 0.01
    sigma = r / 7.0
    delta = r
    lr = 0.01
    
    renderer.build_graph(K, r, sigma, delta, lr)
    renderer.set_requires_grad("vtx_pos")
    renderer.set_requires_grad("normal")
    
    for _ in range(1000):
        renderer.exec_step()
    
    renderer.save_scene("optimized_scene.gltf")
```

### Edge Cases and Pitfalls
*   **Graph Staticity:** The graph rebuild logic relies on `m_graph_static_cnt` reaching specific thresholds (`FAST_REBUILD_COUNT`, `FULL_REBUILD_COUNT`). If these counts are not defined or set incorrectly, the graph may not rebuild when necessary, leading to stale computations [S1].
*   **Dirty Flags:** Variables must be correctly marked as dirty when CPU data changes. Failure to do so will prevent data transfer to the GPU, causing optimization to use stale data [S1].
*   **Gradient Aggregation:** Gradients for the same variable from multiple paths must be summed. Failure to sum gradients correctly will lead to incorrect updates [S1].
*   **Vulkan Command Buffer:** The command buffer must be properly submitted and reset. Failure to submit or reset can lead to GPU hangs or incorrect execution order [S1].

## Operating characteristics
*   **Time Complexity:** The forward pass complexity depends on the rendering algorithm (HardSoftRas) and the number of samples. The backward pass complexity is proportional to the forward pass, as it traverses the computational graph [S1]. The optimization loop runs for a fixed number of iterations (e.g., 1000) [S1].
*   **Memory Complexity:** Memory usage is dominated by the storage of the computational graph and intermediate variables. DressiAD aims to reduce memory usage by packing the graph into Vulkan stages and only rebuilding when necessary, but exact complexity is not given [S1].
*   **Performance:** Performance is hardware-dependent due to the use of Vulkan. The system is designed to be hardware-agnostic [S1].
*   **Strengths:** Define-and-run approach simplifies implementation. Integration with Vulkan allows for efficient GPU execution. Reactive shader packing optimizes for specific rendering scenarios [S1].
*   **Limitations:** Requires manual graph construction. Limited to the functions and operators provided by the system. Debugging can be difficult due to the compiled nature of the Vulkan commands [S1].

## Background
The DressiAD Optimization Loop is part of the Dressi renderer, a hardware-agnostic differentiable renderer with reactive shader packing and soft rasterization [S1]. It was introduced in the paper "Dressi: A Hardware‐Agnostic Differentiable Renderer with Reactive Shader Packing and Soft Rasterization" [S1]. The system is designed to be similar to common AD libraries like PyTorch and TensorFlow but is optimized for differentiable rendering [S1].

## Sources
[S1] Dressi: A Hardware‐Agnostic Differentiable Renderer with Reactive Shader Packing and Soft Rasterization, https://doi.org/10.1111/cgf.14455, fetched 2026-10-29
[S2] MRD: Using Physically Based Differentiable Rendering to Probe Vision Models for 3D Scene Understanding, https://doi.org/10.48550/arXiv.2512.12307, fetched 2026-10-29 (unofficial)
