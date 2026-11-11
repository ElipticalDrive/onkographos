# Checkpointing for Reverse-Mode Memory Optimization

Also called: Checkpointing, Griewank checkpointing, Logarithmic checkpointing, re-evaluation from checkpoints

## Overview
Checkpointing is an optimization technique for reverse-mode automatic differentiation (AD) that reduces the memory complexity of computing gradients by storing only a sparse subset of intermediate variables (checkpoints) and recomputing the loop segments between them during the backward pass [S1, S2]. It addresses the scalability issue of exhaustive storage, where memory usage is proportional to the number of instructions or loop length, by trading increased time complexity for logarithmic memory usage [S1, S2]. Unlike [Inversion Trick](964-inversion-trick.md) or [Path Replay Backpropagation](6780-path-replay-backpropagation.md), which exploit problem-specific structures to avoid differentiation entirely, checkpointing is a general-purpose mechanism applicable to any traced computation [S2].

## How it works
**Inputs:** A primal computation consisting of $N$ sequential operations (or a loop of length $N$) and a set of checkpoint indices.
**Outputs:** Gradients of the scalar loss with respect to the input parameters.

**Data Structures:**
1.  **Checkpoint Stack:** A list or array storing the full state (intermediate variables) at specific indices. The size is $O(\log N)$.
2.  **Primal State:** The current state of variables during recomputation.

**Algorithm:**
The method divides the sequence of $N$ operations into segments. During the forward pass, the state is saved at predefined checkpoint indices. During the backward pass, the algorithm iterates backwards through the segments. For each segment, it restores the state from the nearest preceding checkpoint and re-executes the forward operations of that segment to reconstruct the intermediate values required for gradient computation.

1.  **Forward Pass with Checkpointing:**
    *   Initialize the primal state $x_0$.
    *   Iterate $i$ from $0$ to $N-1$.
    *   Compute $x_{i+1} = f_i(x_i)$.
    *   If $i$ is a checkpoint index, save $x_{i+1}$ to the Checkpoint Stack.
    *   *Note:* The optimal strategy for selecting checkpoint indices to minimize total recomputation time while maintaining $O(\log N)$ memory is to use a binary tree structure or recursive doubling [S1, S2].

2.  **Backward Pass with Recomputation:**
    *   Initialize gradient accumulator $\bar{x}_N$ (typically $\partial L / \partial x_N$).
    *   Iterate backwards through the segments defined by the checkpoints. Let the checkpoints be at indices $c_0, c_1, \dots, c_k$ where $c_k = N$.
    *   For each segment from $c_j$ to $c_{j+1}$:
        *   Restore the primal state $x_{c_j}$ from the Checkpoint Stack.
        *   Recompute the forward operations for indices $i$ from $c_j$ to $c_{j+1}-1$:
            *   $x_{i+1} = f_i(x_i)$.
        *   Compute gradients for this segment in reverse order ($i$ from $c_{j+1}-1$ down to $c_j$):
            *   $\bar{x}_i = \bar{x}_{i+1} \cdot \frac{\partial f_i}{\partial x_i}$.
            *   Accumulate $\bar{x}_i$ into the global gradient accumulator.
    *   *Note:* If the computation involves recursive function calls, a tree structure may be used instead of a stack to store intermediate results [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def checkpointed_backward(N, f, f_prime, checkpoints, x0, grad_out):
    """
    N: Total number of operations.
    f: Function computing next state from current state.
    f_prime: Function computing local gradient (adjoint) of f.
    checkpoints: Sorted list of indices where state is saved.
    x0: Initial state.
    grad_out: Gradient of loss with respect to final state.
    """
    # Forward pass: save checkpoints
    state = x0
    saved_states = {}
    for i in range(N):
        state = f(state)
        if i in checkpoints:
            saved_states[i] = state
    
    # Backward pass: recompute and differentiate
    grad = grad_out
    # Iterate backwards through segments
    # Determine segment boundaries from checkpoints
    # Example: if checkpoints = [0, N/2, N], segments are [0, N/2] and [N/2, N]
    # We need to process segments in reverse order of their end indices
    
    segment_ends = sorted(checkpoints + [N], reverse=True)
    
    for j in range(len(segment_ends) - 1):
        end_idx = segment_ends[j]
        start_idx = segment_ends[j+1]
        
        # Restore state from the checkpoint at start_idx
        # If start_idx is 0, use x0
        if start_idx == 0:
            current_state = x0
        else:
            # The checkpoint is saved at the index *after* the operation
            # So if we checkpointed at index k, we saved state after f_k
            # Here start_idx is the index of the first operation in the segment
            # We need the state *before* start_idx.
            # If start_idx is in checkpoints, saved_states[start_idx-1] holds state after f_{start_idx-1}
            # Wait, standard definition: checkpoint at i means save state after f_i.
            # So to start segment [start_idx, end_idx), we need state after f_{start_idx-1}.
            # If start_idx == 0, state is x0.
            # If start_idx > 0, we look for checkpoint at start_idx - 1.
            if start_idx - 1 in saved_states:
                current_state = saved_states[start_idx - 1]
            else:
                # This case implies a gap in checkpoints not covered by the O(log N) strategy
                # In a proper binary tree strategy, every segment start is a checkpoint.
                raise ValueError("Missing checkpoint for segment start")
        
        # Recompute forward and compute gradients in reverse
        # We need to store intermediate states for gradient computation?
        # No, we can compute gradients on the fly if we have the forward pass values?
        # Actually, standard reverse mode requires the forward values to compute the adjoint.
        # So we must re-run the forward pass for the segment to get x_i, then compute grad.
        
        # Store forward values for this segment to use in backward step
        forward_values = []
        temp_state = current_state
        for i in range(start_idx, end_idx):
            forward_values.append(temp_state)
            temp_state = f(temp_state)
        
        # Now backward through this segment
        # The state after the last operation in the segment is temp_state
        # But we already have grad for the end of the segment from the previous iteration
        # Wait, the loop above processes segments from end to start.
        # So 'grad' is the gradient at the end of the current segment.
        
        for i in range(end_idx - 1, start_idx - 1, -1):
            # x_i is forward_values[i - start_idx]
            # x_{i+1} is forward_values[i - start_idx + 1] if i < end_idx - 1 else temp_state?
            # Actually, forward_values[i-start_idx] is x_i.
            # We need x_{i+1} to compute the adjoint if f_prime depends on it?
            # Usually f_prime(x_i, x_{i+1}) or just x_i.
            # Let's assume f_prime takes current state and produces gradient w.r.t input.
            # Standard adjoint: grad_in = grad_out * df/dx_in.
            
            x_i = forward_values[i - start_idx]
            # If i is the last element, x_{i+1} is the state after the segment.
            # But we don't store x_{i+1} explicitly in forward_values for the last step?
            # Let's assume f_prime can compute gradient from x_i and the gradient flowing out.
            
            grad = f_prime(x_i, grad)
            
        # Update the gradient for the start of this segment to be passed to the previous segment
        # The gradient at start_idx-1 is now in 'grad'
        # But we need to save it? No, it flows to the previous segment.
        # However, if start_idx > 0, we need to ensure the gradient at start_idx-1 is correct.
        # The loop continues to the previous segment.
        
    return grad
```
*Note: The pseudocode above is a simplified representation. A robust implementation uses a binary tree of checkpoints to minimize recomputation steps [S1, S2]. The exact number of recomputations depends on the tree structure.*

**Parameters:**
*   **Checkpoint Frequency:** The number of checkpoints $K$. To achieve $O(\log N)$ memory, $K \approx \log_2 N$.
*   **Checkpoint Indices:** Typically chosen using a recursive doubling strategy (e.g., indices $N/2, N/4, 3N/4, \dots$) [S1, S2].

**Edge Cases and Pitfalls:**
*   **Precision:** Recomputing operations can introduce floating-point rounding errors that differ from the forward pass, potentially affecting gradient accuracy [S1].
*   **Memory vs. Time Trade-off:** While memory is reduced to $O(\log N)$, time complexity increases to $O(N \log N)$. For small $N$, exhaustive storage may be faster [S1].
*   **Recursive Calls:** If the function $f$ calls itself, the checkpointing strategy must handle the call stack correctly, potentially using a tree structure instead of a linear stack [S1].

## Subtopics

### Custom Adjoint Method
Differentiation techniques that exploit problem-specific traits to reduce storage and reevaluation overheads, generally preferable to checkpointing when available [S2]. For Newton's method, the implicit function theorem provides the answer directly from the iteration's fixed point instead of differentiating through all steps [S2]. For ordinary differential equations, reverse time integration provides an efficient adjoint [S2].

### Checkpointing via JIT Layer
A checkpointing mechanism that arises naturally from the combined AD/JIT system without intentional implementation effort. The JIT layer recomputes intermediate steps and temporaries from the last evaluation point [S2]. In systems like Dr.Jit, calling `dr.backward(data)` causes the AD layer to reference intermediate steps produced by `f`, which the JIT layer recomputes from the last `dr.eval` checkpoint [S2]. Checkpointing is not ideal for Path Replay Backpropagation (PBDR) workloads due to the large size of even a small number of checkpoints [S2].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$, where $N$ is the number of operations or loop length [S1, S2]. This is higher than the $O(N)$ time of exhaustive storage but acceptable for large $N$.
*   **Memory Complexity:** $O(\log N)$, where $N$ is the number of operations [S1, S2]. This is a significant reduction from the $O(N)$ memory of exhaustive storage.
*   **Scaling:** The method scales well with increasing $N$ in terms of memory, making it suitable for large-scale differentiable rendering where intermediate state can be terabytes in size [S2].
*   **Strengths:** General applicability to any traced computation; significant memory savings for long sequences of operations.
*   **Limitations:** Increased computational overhead due to recomputation; potential for precision errors; not ideal for workloads with large checkpoint sizes (e.g., PBDR) [S2].
*   **When to Use:** When memory is the bottleneck and the computation is too large for exhaustive storage, and no custom adjoint is available.
*   **When Not to Use:** When custom adjoints are available (e.g., [Inversion Trick](964-inversion-trick.md), [Path Replay Backpropagation](6780-path-replay-backpropagation.md)), or when $N$ is small enough that exhaustive storage fits in memory.

## Background
Checkpointing for reverse-mode automatic differentiation was introduced by Andreas Griewank in 1992 [S1]. The technique addresses the "age-old question" of how to manage intermediate state in reversed program references, where exhaustive storage does not scale due to the volume of data generated by modern processors [S2]. The concept of storing state at sparse checkpoints and recovering it via reevaluation was previously noted by Volin and Ostrovskii in 1985 [S2]. Automatic recursive usage of this pattern was further analyzed by Siskind and Pearlmutter in 2018, formalizing the $O(\log t)$ overhead for a program with $t$ operations [S2]. In the context of differentiable rendering, checkpointing is a key component of general-purpose AD systems like Dr.Jit, enabling the differentiation of complex rendering programs without requiring custom adjoints for every operation [S2].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S2] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
