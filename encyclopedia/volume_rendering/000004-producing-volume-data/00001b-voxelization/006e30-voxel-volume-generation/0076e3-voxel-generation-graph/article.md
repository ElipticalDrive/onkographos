# Voxel Generation Graph

Also called: Generation Graph Node Evaluation, GenerationGraph, GenerationGraphStack, Voxel Evaluation Graph

## Overview
The Voxel Generation Graph is a stack-based evaluation system for generating voxel data (value, gradient, and material) at a specific 3D position by executing a sequence of node operations defined in a global buffer [S1, unofficial]. It solves the problem of procedural voxel generation by allowing complex volumetric functions to be composed from atomic primitives (noise, transforms, CSG) in a linear, executable graph [S1, unofficial]. It is distinct from direct shader-based evaluation because it decouples the logical structure of the volume definition (the graph) from the evaluation loop, enabling dynamic modification of voxel properties via CSG operations and material assignment at runtime [S1, unofficial].

## How it works
The method evaluates a directed acyclic graph of nodes to produce a single `Voxel` output. The evaluation state is maintained in a fixed-size stack.

### Inputs and Outputs
*   **Input:** A global buffer of `GenerationGraphNode` structures (`generationGraphNodes`) and an integer `numberOfGenerationGraphNodes` defining the graph size. The evaluation is triggered for a specific 3D position (typically the world position of a chunk or voxel).
*   **Output:** A single `Voxel` structure containing the final value, gradient, and material index.

### Data Structures
The implementation relies on three primary structures defined in the source [S1, unofficial]:

1.  **`GenerationGraphNode`**: Defines a single operation in the graph.
    *   `uint nodeType`: Enum value indicating the operation (see NodeType constants below).
    *   `float4x4 transformMatrix`: Used if `nodeType` is Transform.
    *   `NoiseParameters noiseParameters`: Used if `nodeType` is Noise or DomainWarp.
    *   `CSGPrimitive csgPrimitive`: Used if `nodeType` is CSGPrimitive.
    *   `uint materialIndex`: Used if `nodeType` is Material.
    *   `CSGOperator csgOperator`: Used if `nodeType` is CSGOperation.

2.  **`GenerationGraphStack`**: The evaluation stack.
    *   `Voxel buffer[2]`: A fixed buffer holding up to 2 voxels.
    *   `uint count`: The current number of voxels on the stack.
    *   *Note:* The stack size is strictly limited to 2 voxels [S1, unofficial].

3.  **`Voxel`**: The data unit pushed/popped from the stack.
    *   `float4 valueAndGradient`: Contains the scalar value and its 3D gradient.
    *   Additional gradient/position data may be present depending on the specific `Voxel` implementation, but the core output is the value/gradient pair [S1, unofficial].

4.  **`GPUVoxelVolumeCSGOperation`**: (C# struct, sequential layout) Used to apply CSG operations to existing voxel volumes.
    *   `GPUCSGOperator CSGOperator`
    *   `GPUCSGPrimitive CSGPrimitive`
    *   `MaterialIndex MaterialIndex`
    *   `Matrix4x4 TransformMatrix`

### NodeType Constants
The `nodeType` field in `GenerationGraphNode` uses the following integer values [S1, unofficial]:
*   `Position`: 0
*   `Transform`: 1
*   `DomainWarp`: 2
*   `Noise`: 3
*   `CSGPrimitive`: 4
*   `Material`: 5
*   `CSGOperation`: 6
*   `Output`: 7

### Evaluation Steps
The evaluation iterates through the `generationGraphNodes` buffer from index 0 to `numberOfGenerationGraphNodes - 1`. For each node, the corresponding operation is performed on the stack.

1.  **Initialize Stack**: Create an empty `GenerationGraphStack` with `count = 0` and `buffer` cleared.
2.  **Iterate Nodes**: For `i` from 0 to `numberOfGenerationGraphNodes - 1`:
    *   Let `node` be `generationGraphNodes[i]`.
    *   Switch on `node.nodeType`:
        *   **Position (0)**: Push the current evaluation position onto the stack.
            *   *Implementation Note:* The "current evaluation position" is typically the input position passed to the graph evaluation function. If multiple Position nodes exist, they likely refer to the same base position or a context-specific coordinate [S1, unofficial].
        *   **Transform (1)**:
            *   Pop `pos` from the stack.
            *   Compute `new_pos = node.transformMatrix * pos`.
            *   Push `new_pos` onto the stack.
        *   **DomainWarp (2)**:
            *   Pop `pos` from the stack.
            *   Compute `warped_pos = WarpDomain(pos, node.noiseParameters)`.
            *   Push `warped_pos` onto the stack.
        *   **Noise (3)**:
            *   Pop `pos` from the stack.
            *   Compute `value` and `gradient` using Fractal Brownian Motion (FBM) noise with `node.noiseParameters` at `pos`.
            *   Create a `Voxel` with `valueAndGradient = float4(value, gradient.x, gradient.y, gradient.z)`.
            *   Push the `Voxel` onto the stack.
        *   **CSGPrimitive (4)**:
            *   Pop `pos` from the stack.
            *   Evaluate the signed distance function (SDF) of `node.csgPrimitive` at `pos` to get `value` and `gradient`.
            *   Create a `Voxel` with `valueAndGradient`.
            *   Push the `Voxel` onto the stack.
        *   **Material (5)**:
            *   Pop `voxel` from the stack.
            *   Create a new `Voxel` with the same `valueAndGradient` as `voxel`, but set its material index to `node.materialIndex`.
            *   Push the new `Voxel` onto the stack.
        *   **CSGOperation (6)**:
            *   Pop `rhs` from the stack.
            *   Pop `lhs` from the stack.
            *   *Note:* The right-hand side is popped first to maintain standard left-hand-side operator precedence (lhs op rhs) [S1, unofficial].
            *   Compute `result_voxel = ApplyCSG(node.csgOperator, lhs, rhs)`.
            *   Push `result_voxel` onto the stack.
        *   **Output (7)**:
            *   Pop `final_voxel` from the stack.
            *   Return `final_voxel`.
3.  **Return**: If the loop completes without hitting an Output node, the behavior is undefined or returns the top of the stack [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_voxel_generation_graph(generation_graph_nodes, number_of_nodes, initial_position):
    """
    Evaluates a voxel generation graph to produce a single Voxel.
    
    Args:
        generation_graph_nodes: List of GenerationGraphNode structs.
        number_of_nodes: Integer count of nodes.
        initial_position: float3, the starting 3D position for evaluation.
        
    Returns:
        Voxel: The final voxel result.
    """
    
    # Stack implementation
    stack_buffer = [None, None]  # Fixed size 2
    stack_count = 0
    
    def push_voxel(v):
        nonlocal stack_count
        if stack_count >= 2:
            raise ValueError("Stack overflow: GenerationGraphStack is limited to 2 voxels")
        stack_buffer[stack_count] = v
        stack_count += 1
        
    def pop_voxel():
        nonlocal stack_count
        if stack_count <= 0:
            raise ValueError("Stack underflow")
        stack_count -= 1
        return stack_buffer[stack_count]

    for i in range(number_of_nodes):
        node = generation_graph_nodes[i]
        node_type = node.nodeType
        
        if node_type == 0:  # Position
            # Push current evaluation position. 
            # In a simple graph, this is the initial_position. 
            # If the graph allows multiple positions, context is needed.
            # Assuming the graph expects a position input at this stage.
            # We push a dummy voxel or just the position? 
            # The spec says "Push current evaluation position onto stack".
            # However, subsequent nodes (Noise, CSG) expect a Voxel or Position?
            # Noise pops position and pushes Voxel. CSG pops position and pushes Voxel.
            # Transform pops position and pushes position.
            # So the stack holds mixed types? Or is it always Voxel?
            # The spec says GenerationGraphStack has a buffer of 2 Voxels.
            # This implies Position nodes might push a Voxel with position data?
            # Or the stack is polymorphic. Given "Voxel buffer[2]", let's assume
            # Position nodes push a Voxel where valueAndGradient encodes the position?
            # Or more likely, the "Position" node sets the *current* position context,
            # and other nodes read from that context?
            # Re-reading: "Pop position from stack". This implies the stack holds positions.
            # But "Push result back" for Transform.
            # And Noise pushes "result" (Voxel).
            # This suggests the stack holds heterogeneous data or the "Voxel" struct
            # can represent a position.
            # Let's assume the stack holds generic data items, but the struct is named Voxel.
            # For implementation clarity, we will treat the stack as holding "EvaluationItems".
            # If the source strictly says "Voxel buffer", we assume a Voxel struct is used
            # to represent positions (e.g., value=0, gradient=position).
            
            # Implementation: Push a Voxel representing the position.
            # Value=0, Gradient=initial_position (or current context position).
            # Since Position node is usually at the start, we use initial_position.
            pos_voxel = Voxel(value=0.0, gradient=initial_position)
            push_voxel(pos_voxel)
            
        elif node_type == 1:  # Transform
            item = pop_voxel()
            # Item is a position (Voxel with value=0)
            pos = item.gradient
            new_pos = transform_matrix_multiply(node.transformMatrix, pos)
            new_voxel = Voxel(value=0.0, gradient=new_pos)
            push_voxel(new_voxel)
            
        elif node_type == 2:  # DomainWarp
            item = pop_voxel()
            pos = item.gradient
            warped_pos = warp_domain(pos, node.noiseParameters)
            new_voxel = Voxel(value=0.0, gradient=warped_pos)
            push_voxel(new_voxel)
            
        elif node_type == 3:  # Noise
            item = pop_voxel()
            pos = item.gradient
            value, gradient = generate_fbm_noise(pos, node.noiseParameters)
            new_voxel = Voxel(value=value, gradient=gradient)
            push_voxel(new_voxel)
            
        elif node_type == 4:  # CSGPrimitive
            item = pop_voxel()
            pos = item.gradient
            value, gradient = evaluate_csg_primitive(node.csgPrimitive, pos)
            new_voxel = Voxel(value=value, gradient=gradient)
            push_voxel(new_voxel)
            
        elif node_type == 5:  # Material
            voxel = pop_voxel()
            # Create new voxel with same value/gradient but new material
            new_voxel = Voxel(
                value=voxel.value, 
                gradient=voxel.gradient, 
                material_index=node.materialIndex
            )
            push_voxel(new_voxel)
            
        elif node_type == 6:  # CSGOperation
            rhs = pop_voxel()
            lhs = pop_voxel()
            result_voxel = apply_csg_operation(node.csgOperator, lhs, rhs)
            push_voxel(result_voxel)
            
        elif node_type == 7:  # Output
            final_voxel = pop_voxel()
            return final_voxel
            
    # If no Output node is reached, return the top of the stack or error
    if stack_count > 0:
        return pop_voxel()
    else:
        raise ValueError("Graph did not produce an output")

def transform_matrix_multiply(matrix, position):
    # Homogeneous coordinate multiplication
    x, y, z = position
    w = 1.0
    # matrix is float4x4, stored as 4 rows or columns depending on convention
    # Assuming row-major or standard math library convention
    # result = matrix * float4(x,y,z,1)
    # Simplified for pseudocode:
    return float3(
        matrix[0][0]*x + matrix[1][0]*y + matrix[2][0]*z + matrix[3][0],
        matrix[0][1]*x + matrix[1][1]*y + matrix[2][1]*z + matrix[3][1],
        matrix[0][2]*x + matrix[1][2]*y + matrix[2][2]*z + matrix[3][2]
    )

def warp_domain(position, noise_params):
    # Apply domain warping using noise
    # This is a placeholder for the specific warping algorithm
    # Typically: warped = position + noise(position) * scale
    return position  # Placeholder

def generate_fbm_noise(position, noise_params):
    # Generate Fractal Brownian Motion
    # Returns (value, gradient)
    return 0.0, float3(0,0,0) # Placeholder

def evaluate_csg_primitive(primitive, position):
    # Evaluate SDF and gradient
    return 0.0, float3(0,0,0) # Placeholder

def apply_csg_operation(operator, lhs, rhs):
    # Apply CSG operator (Union, Intersection, Difference)
    # Returns new Voxel
    return lhs # Placeholder

class Voxel:
    def __init__(self, value, gradient, material_index=0):
        self.value = value
        self.gradient = gradient
        self.material_index = material_index
```

### Parameters and Typical Values
*   **Stack Size**: Fixed at 2 voxels [S1, unofficial].
*   **NodeType Enum**: Integers 0-7 [S1, unofficial].
*   **Noise Parameters**: Specific to the noise implementation (octaves, lacunarity, gain, etc.), not detailed in the source [S1, unofficial].
*   **CSG Operators**: Standard CSG operations (Union, Intersection, Difference) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Stack Overflow**: The stack is limited to 2 voxels. Any operation requiring more than 2 operands must be structured as sequential binary operations (e.g., `A op B op C` must be `(A op B) op C`) [S1, unofficial].
*   **Operand Order**: For `CSGOperation`, the right-hand side (RHS) is popped before the left-hand side (LHS) to ensure correct operator precedence (`lhs op rhs`) [S1, unofficial].
*   **Missing Output**: If the graph does not end with an `Output` node, the result is undefined. The implementation should handle this gracefully, either by returning the top of the stack or raising an error [S1, unofficial].
*   **Type Safety**: The stack holds heterogeneous data (positions and voxels). The implementation must ensure that nodes pop the correct type of data. For example, `Transform` expects a position, while `Noise` expects a position but produces a voxel [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$, where $N$ is the number of nodes in the generation graph. Each node is evaluated exactly once [S1, unofficial].
*   **Memory Complexity**: $O(1)$ for the stack (fixed size 2) plus $O(N)$ for the graph node buffer. The memory footprint is dominated by the `generationGraphNodes` buffer [S1, unofficial].
*   **Performance**: Performance depends on the complexity of the individual node operations (e.g., noise generation, CSG evaluation). The stack-based approach is efficient for GPU execution as it minimizes branching and allows for tight loops [S1, unofficial].
*   **Strengths**:
    *   Dynamic graph construction allows for runtime modification of voxel volumes.
    *   Stack-based evaluation is simple to implement and debug.
    *   Supports complex compositions of noise, transforms, and CSG [S1, unofficial].
*   **Limitations**:
    *   Stack size limit of 2 restricts the complexity of single-step operations.
    *   Linear evaluation order may not be optimal for all graph structures (e.g., if nodes are independent, parallel evaluation could be faster) [S1, unofficial].
*   **When to Use**: Suitable for procedural voxel generation in real-time applications where dynamic modification of voxel properties is required, such as in the Tuntenfisch/Voxels project [S1, unofficial].

## Background
The Voxel Generation Graph was introduced in the **Tuntenfisch/Voxels** project, an open-source voxel engine for Unity. It is used to generate procedural voxel volumes by evaluating a graph of nodes that define the volume's properties (noise, CSG, materials) at runtime [S1, unofficial]. The method is part of a larger system that includes dual contouring for mesh generation and chunk-based world management [S1, unofficial].

## Sources
[S1] Tantenfisch/Voxels, https://github.com/Tantenfisch/Voxels, fetched 2026-10-26 (unofficial)
