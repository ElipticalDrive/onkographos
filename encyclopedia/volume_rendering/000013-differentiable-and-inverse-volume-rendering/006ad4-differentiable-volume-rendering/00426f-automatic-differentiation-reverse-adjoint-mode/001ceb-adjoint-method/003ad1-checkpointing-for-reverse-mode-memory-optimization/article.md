# Checkpointing for Reverse-Mode Memory Optimization

Also called: Checkpointing, Griewank checkpointing, Logarithmic checkpointing, re-evaluation from checkpoints

## Overview
Checkpointing is an optimization for reverse-mode automatic differentiation (AD) that reduces memory complexity by storing only a sparse subset of intermediate variables and recomputing loop segments during the backward pass [S1, S2]. It addresses the scalability issue of exhaustive storage, where memory usage is proportional to the number of instructions, by trading increased time complexity for logarithmic memory usage [S1, S2]. Unlike [Inversion Trick](964-inversion-trick.md) or [Path Replay Backpropagation](6780-path-replay-backpropagation.md), which exploit problem-specific structures to avoid differentiation entirely, checkpointing is a general-purpose mechanism applicable to any traced computation [S2].

## How it works
**Inputs:** A primal computation consisting of $N$ sequential operations (or a loop of length $N$) [S1, S2].
**Outputs:** Gradients of a scalar loss with respect to the input parameters [G].

**Data Structures:**
1. **Checkpoint Storage:** A dictionary or array storing the full state at specific indices. The size is $O(\log N)$ [S2].
2. **Primal State:** The current state of variables during recomputation [G].

**Algorithm:**
The method divides the sequence of $N$ operations into segments [G]. During the forward pass, the state is saved at predefined checkpoint indices [G]. During the backward pass, the algorithm iterates backwards through the segments [G]. For each segment, it restores the state from the nearest preceding checkpoint and re-executes the forward operations to reconstruct intermediate values required for gradient computation [S1, S2].

1. **Forward Pass with Checkpointing:**
   * Initialize the primal state $x_0$ [G].
   * Iterate $i$ from $0$ to $N-1$ [G].
   * Compute $x_{i+1} = f_i(x_i)$ [G].
   * If $i$ is a checkpoint index, save $x_i$ to the Checkpoint Storage [G].
   * *Note:* The optimal strategy for selecting checkpoint indices to minimize total recomputation time while maintaining $O(\log N)$ memory is to use a recursive doubling or tree strategy [S1, S2].

2. **Backward Pass with Recomputation:**
   * Initialize gradient accumulator $\bar{x}_N$ (typically $\partial L / \partial x_N$) [G].
   * Iterate backwards through the segments defined by the checkpoints. Let the checkpoints be at indices $c_0, c_1, \dots, c_k$ where $c_0 = 0$ and $c_k = N$ [G].
   * For each segment from $c_j$ to $c_{j+1}$:
     * Restore the primal state $x_{c_j}$ from the Checkpoint Storage [G].
     * Recompute the forward operations for indices $i$ from $c_j$ to $c_{j+1}-1$: $x_{i+1} = f_i(x_i)$ [G].
     * Compute gradients for this segment in reverse order ($i$ from $c_{j+1}-1$ down to $c_j$): $\bar{x}_i = \bar{x}_{i+1} \cdot \frac{\partial f_i}{\partial x_i}$ [G].
     * Accumulate $\bar{x}_i$ into the global gradient accumulator [G].
   * *Note:* If the computation involves recursive function calls, a tree structure may be used instead of a stack to store intermediate results [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def checkpointed_backward(N, f, f_prime, checkpoints, x0, grad_out):
    saved_states = {}
    state = x0
    # Forward pass: save state before update to enable correct segment restoration
    for i in range(N):
        if i in checkpoints:
            saved_states[i] = state
        state = f(state)
        
    grad = grad_out
    # Sort checkpoint boundaries in descending order to process segments backwards
    segment_ends = sorted(checkpoints + [N], reverse=True)
    
    for j in range(len(segment_ends) - 1):
        end_idx = segment_ends[j]
        start_idx = segment_ends[j+1]
        
        # Restore primal state at segment start; default to x0 if start_idx is 0
        current_state = saved_states.get(start_idx, x0)
        forward_states = [current_state]
        temp_state = current_state
        
        # Recompute forward operations for the segment
        for i in range(start_idx, end_idx):
            temp_state = f(temp_state)
            forward_states.append(temp_state)
            
        # Compute gradients in reverse order for the segment
        for i in range(end_idx - 1, start_idx - 1, -1):
            x_i = forward_states[i - start_idx]
            x_next = forward_states[i - start_idx + 1]
            # Standard reverse-mode adjoint update: xbar_i = xbar_next * f'(x_i)
            grad = f_prime(x_i, x_next, grad)
            
    return grad
```

**Parameters:**
* **Checkpoint Frequency:** The number of checkpoints $K$. To achieve $O(\log N)$ memory, $K \approx \log_2 N$ [S1, S2].

**Edge Cases and Pitfalls:**
* **Recursive Calls:** If the function $f$ calls itself, the checkpointing strategy must handle the call stack correctly, potentially using a tree structure instead of a linear stack [S1].
* **Memory vs. Time Trade-off:** While memory is reduced to $O(\log N)$, time complexity increases to $O(N \log N)$ [S1, S2].

## Subtopics

### Custom Adjoint Method
Differentiation techniques that exploit problem-specific traits to reduce storage and reevaluation overheads, generally preferable to checkpointing when available [S2]. For Newton's method, the implicit function theorem provides the answer directly from the iteration's fixed point instead of differentiating through all steps [S2]. For ordinary differential equations, reverse time integration provides an efficient adjoint [S2].

### Checkpointing via JIT Layer
A checkpointing mechanism that arises naturally from the combined AD/JIT system without intentional implementation effort [S2]. The JIT layer recomputes intermediate steps and temporaries from the last evaluation point [S2]. In systems like Dr.Jit, calling `dr.backward(data)` causes the AD layer to reference intermediate steps produced by `f`, which the JIT layer recomputes from the last `dr.eval` checkpoint [S2]. Checkpointing is not ideal for Path Replay Backpropagation (PBDR) workloads due to the large size of even a small number of checkpoints [S2].

## Operating characteristics
* **Time Complexity:** $O(N \log N)$, where $N$ is the number of operations or loop length [S1, S2].
* **Memory Complexity:** $O(\log N)$, where $N$ is the number of operations [S1, S2].
* **Scaling:** Reduces memory complexity to $O(\log N)$, addressing scalability issues where exhaustive storage generates terabytes of intermediate state [S2].
* **Strengths:** General applicability to any traced computation; significant memory savings for long sequences of operations [S2, G].
* **Limitations:** Increased computational overhead due to recomputation; not ideal for workloads with large checkpoint sizes (e.g., PBDR) [S2].
* **When to Use:** When memory is the bottleneck and the computation is too large for exhaustive storage, and no custom adjoint is available [S2].
* **When Not to Use:** When custom adjoints are available (e.g., [Inversion Trick](964-inversion-trick.md), [Path Replay Backpropagation](6780-path-replay-backpropagation.md)) [S2].

## Background
Checkpointing for reverse-mode automatic differentiation was introduced by Andreas Griewank in 1992 [S1]. The technique addresses the "age-old question" of how to manage intermediate state in reversed program references, where exhaustive storage does not scale due to the volume of data generated by modern processors [S2]. The concept of storing state at sparse checkpoints and recovering it via reevaluation was previously noted by Volin and Ostrovskii in 1985 [S2]. Automatic recursive usage of this pattern was further analyzed by Siskind and Pearlmutter in 2018, formalizing the $O(\log t)$ overhead for a program with $t$ operations [S2]. In the context of differentiable rendering, checkpointing is a key component of general-purpose AD systems like Dr.Jit, enabling the differentiation of complex rendering programs without requiring custom adjoints for every operation [S2].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S2] DR.JIT, https://doi.org/10.1145/3528223.3530099, fetched 2026-10-26
