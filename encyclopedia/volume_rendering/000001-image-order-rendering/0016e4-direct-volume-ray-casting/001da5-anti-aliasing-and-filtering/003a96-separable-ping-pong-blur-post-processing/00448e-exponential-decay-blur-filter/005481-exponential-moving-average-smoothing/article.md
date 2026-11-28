# Exponential Moving Average Smoothing

Also called: Control Signal Smoothing, EMA Smoothing, Exponential Smoothing

## Overview
Exponential Moving Average (EMA) smoothing recursively estimates a signal by combining the current observation with the previous smoothed value [S1, unofficial][S2]. It is distinguished from fixed-kernel blurs by its infinite impulse response, where the effective memory window is determined by the smoothing parameter $\alpha$ rather than a fixed spatial or temporal radius [G].

## How it works
**Inputs and Outputs**
- Input: A sequence of raw values $v_i$ for $i = 1, \dots, N$.
- Output: A sequence of smoothed values $\hat{v}_i$.
- Parameter: Smoothing factor $\alpha \in (0, 1]$.

**Data Structures**
- `v`: Array of raw input values.
- `v_hat`: Array of smoothed output values, same size as `v`.
- `alpha`: Scalar float.

**Formula**
$$
\hat{v}_i = \begin{cases} 
v_1 & i = 1 \\
\alpha v_i + (1 - \alpha) \hat{v}_{i-1} & i > 1
\end{cases}
$$
where $v_i$ is the current raw input value, $\hat{v}_{i-1}$ is the previously computed smoothed value, and $\alpha$ is the smoothing parameter [S1, unofficial].

**Steps**
1. Initialize $\hat{v}_1 = v_1$.
2. For $i$ from 2 to $N$: compute $\hat{v}_i = \alpha v_i + (1 - \alpha) \hat{v}_{i-1}$.
3. Return $\hat{v}$.

**Parameters**
- $\alpha$: Controls the trade-off between smoothing and responsiveness. Smaller $\alpha$ results in more smoothing; larger $\alpha$ results in less dependence on previous values and thus less smoothing [S1, unofficial]. Typical value: $\alpha = 0.8$ is used for general methods and control signal filtering [S1, unofficial][S2]. Specific case: $\alpha = 0.3$ is used for IDR due to higher variance between plot points [S1, unofficial].

**Edge Cases and Pitfalls**
- **Initialization**: The value at $i=1$ uses the raw value directly to avoid bias [S1, unofficial].
- **Precision**: Not given in the sources.
- **Boundary**: The formula is undefined for $i=1$ without an initial condition, which is resolved by the base case [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exponential_moving_average_smoothing(v, alpha):
    if not v:
        return []
    n = len(v)
    v_hat = [0.0] * n
    v_hat[0] = v[0]
    for i in range(1, n):
        v_hat[i] = alpha * v[i] + (1 - alpha) * v_hat[i-1]
    return v_hat
```

## Subtopics

### Exponential Decay Blur Filter
The parent method that applies this smoothing along the outermost two dimensions of a tensor. It is chosen when spatial filtering along the outer dimensions is required for image-order rendering. See [Exponential Decay Blur Filter](11279-exponential-decay-blur-filter.md).

## Operating characteristics
- **Time and Memory Complexity**: Not given in the sources.
- **Measured Performance and Scaling**: Not given in the sources.
- **Strengths and Limitations**: Not given in the sources.
- **When to use and when not**: Not given in the sources.

## Background
The method originates from statistical smoothing techniques and has been applied in control systems and neural network training [S1, unofficial][S2]. In volume rendering and robotics, it smooths control signals for robot actuation [S2] and reduces variance in training metrics for neural lumigraph representations [S1, unofficial]. The specific parameter values ($\alpha=0.8$ and $\alpha=0.3$) are empirically derived from these applications [S1, unofficial][S2].

## Sources
[S1] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
[S2] NeRF2Real: Sim2real Transfer of Vision-guided Bipedal Motion Skills using Neural Radiance Fields, https://doi.org/10.1109/icra48891.2023.10161544, fetched 2026-10-29
