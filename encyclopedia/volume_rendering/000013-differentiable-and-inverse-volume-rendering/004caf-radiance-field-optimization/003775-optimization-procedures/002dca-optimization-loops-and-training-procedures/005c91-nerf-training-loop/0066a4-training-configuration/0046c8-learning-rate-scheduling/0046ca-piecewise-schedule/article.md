# Piecewise Schedule


## Overview
A learning rate scheduling method that composes multiple sub-schedules applied sequentially at specified milestone steps [S1, unofficial]. It allows complex training dynamics by chaining distinct optimization behaviors (e.g., warmup, decay, fine-tuning) without requiring a single monolithic formula [S1, unofficial]. It differs from continuous decay schedules like [Exponential Decay Schedules](18486-exponential-decay-schedules.md) by allowing abrupt changes in behavior at discrete boundaries, and from [Annealing Schedules](11839-annealing-schedules.md) by its explicit modular composition of independent schedule objects rather than a single parametric function [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A global training step index $t$ (integer $\ge 0$).
- Output: A scalar learning rate value $lr$.

**Data Structures:**
The schedule object maintains two primary fields:
1. `schedules`: A list of `Schedule` objects. Each object must implement a method `get(step_offset)` that returns a learning rate for a local step offset.
2. `milestones`: A 1D array of integers representing cumulative step counts. `milestones[i]` is the global step index where `schedules[i]` begins. The array is sorted in ascending order.

**Algorithm:**
1. **Initialization:** Construct the `schedules` list and the `milestones` array from configuration. Ensure `milestones` has length $N$, where $N$ is the number of sub-schedules.
2. **Lookup:** Given a global step $t$:
   a. Find the index $idx$ such that `milestones[idx]` is the first milestone greater than $t$. This is typically done using a binary search (e.g., `searchsorted` with `side='right'`) [S1, unofficial].
   b. Determine the base step for the current sub-schedule. If $idx > 0$, the base step is `milestones[idx - 1]`. If $idx == 0$, the base step is 0.
   c. Calculate the local step offset: $t_{local} = t - \text{base\_step}$.
   d. Retrieve the learning rate from the selected sub-schedule: $lr = \text{schedules}[idx].\text{get}(t_{local})$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp

class PiecewiseSchedule:
    def __init__(self, schedules, milestones):
        """
        Args:
            schedules: List of Schedule objects, each having a .get(step) method.
            milestones: 1D array of cumulative step counts where each sub-schedule begins.
        """
        self.schedules = schedules
        self.milestones = jnp.array(milestones)

    def get(self, step):
        """
        Returns the learning rate for the given global step.
        """
        # Find the index of the first milestone strictly greater than step
        # side='right' ensures that if step equals a milestone, we pick the next schedule
        # or the current one depending on boundary definition. 
        # Standard implementation uses side='right' to handle [milestone, next_milestone) intervals.
        idx = jnp.searchsorted(self.milestones, step, side='right')
        
        # Select the schedule corresponding to this interval
        schedule = self.schedules[idx]
        
        # Determine the base step for the current schedule
        # If idx is 0, the base is 0. Otherwise, it is the previous milestone.
        if idx >= 1:
            base_idx = self.milestones[idx - 1]
        else:
            base_idx = 0
            
        # Calculate local step and retrieve learning rate
        local_step = step - base_idx
        return schedule.get(local_step)
```

**Parameters and Edge Cases:**
- **Milestone Alignment:** The `milestones` array must be strictly increasing. If `milestones` is empty, the behavior is undefined [G].
- **Step Boundaries:** The use of `side='right'` in `searchsorted` implies that if $t$ equals `milestones[i]`, it falls into the interval starting at `milestones[i]` (index $i$) if we consider the interval $[milestones[i], milestones[i+1])$. However, `searchsorted(..., side='right')` returns the index after the last occurrence of `step`. If `step` equals `milestones[i]`, it returns $i+1$. This means the schedule at index $i+1$ is used for the step equal to the milestone. Implementers must verify if this boundary behavior matches the intended "start at milestone" semantics. If the intent is that `milestones[i]` starts schedule $i$, `side='left'` might be more appropriate, or the milestones must be offset by 1. The source code uses `side='right'` [S1, unofficial].
- **Precision:** Steps are integers. No floating-point precision issues in step calculation.

## Subtopics

### Two-Phase Training Schedule
A specific application of piecewise scheduling for light field compression networks, consisting of a training phase and a quantization phase [S2]. Both phases use a constant learning rate of 0.01 [S2]. The training phase lasts 12 epochs, where one epoch is defined as all Sampled Aperture Images (SAIs) being used 500 times, with 5 SAIs fed per iteration to calculate averaged loss [S2]. The quantization stage defines one epoch as all SAIs involved 200 times, followed by fine-tuning all consecutive layers for just 1 epoch after quantizing each layer [S2]. This variant is chosen when network convergence requires distinct training and quantization-aware fine-tuning stages [S2]. Hardware: Single Nvidia Titan RTX GPU with 24GB memory; Framework: PyTorch [S2].

## Operating characteristics
- **Time Complexity:** $O(\log N)$ per step lookup, where $N$ is the number of sub-schedules, due to binary search on milestones [S1, unofficial].
- **Memory Complexity:** $O(N)$ to store the milestones array and references to sub-schedules [S1, unofficial].
- **Strengths:** Modular design allows mixing different schedule types (e.g., linear warmup followed by cosine decay) [S1, unofficial]. Precise control over training dynamics at specific milestones.
- **Limitations:** Requires careful tuning of milestone boundaries. Discontinuities in learning rate can occur if sub-schedules do not align at boundaries [G].
- **When to use:** When training requires distinct phases with different optimization behaviors (e.g., warmup, decay, fine-tuning) [S1, unofficial].

## Background
The piecewise schedule structure is implemented in the Nerfies project, a JAX-based implementation of NeRF [S1, unofficial]. It provides a flexible way to manage learning rates in complex training loops [S1, unofficial]. The Two-Phase variant is used in research on kernel-modulated neural representations for light field compression [S2].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S2] Learning Kernel-Modulated Neural Representation for Efficient Light Field Compression, https://hal.science/hal-04627349v1, fetched 2026-10-26
