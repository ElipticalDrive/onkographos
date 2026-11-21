# Exponential Moving Average Smoothing

Also called: Control Signal Smoothing, EMA Smoothing, Exponential Smoothing

## Overview
Exponential Moving Average (EMA) smoothing is a recursive low-pass filter that estimates the underlying signal by weighting the current observation and the previous smoothed estimate. In the context of volume rendering and control signal processing, it reduces high-frequency noise and variance in data streams [S1, unofficial][S2]. It is distinguished from fixed-kernel blurs by its infinite impulse response and adaptive memory, where the effective window size is determined by the smoothing parameter $\alpha$ rather than a fixed spatial or temporal radius.

## How it works
**Inputs and Outputs**
- Input: A sequence of raw values $v_i$ for $i = 1, \dots, N$.
- Output: A sequence of smoothed values $\hat{v}_i$.
- Parameter: Smoothing factor $\alpha \in (0, 1]$.

**Data Structures**
- `v`: Array of raw input values.
- `v_hat`: Array of smoothed output values, same size as `v`.
- `alpha`: Scalar float.

**Algorithm**
The method computes the smoothed value $\hat{v}_i$ using the recurrence relation:
$$
\hat{v}_i = \alpha v_i + (1 - \alpha) \hat{v}_{i-1}
$$
where:
- $v_i$ is the current raw input value.
- $\hat{v}_{i-1}$ is the previously computed smoothed value.
- $\alpha$ is the smoothing parameter.

**Steps**
1. Initialize $\hat{v}_0$. If not specified, $\hat{v}_0 = v_1$ or $0$ [G].
2. For $i$ from 1 to $N$:
   a. Compute $\hat{v}_i = \alpha \cdot v_i + (1 - \alpha) \cdot \hat{v}_{i-1}$.
3. Return $\hat{v}$.

**Parameters**
- $\alpha$: Controls the trade-off between smoothing and responsiveness.
  - Smaller $\alpha$ results in more smoothing [S1, unofficial].
  - Larger $\alpha$ results in less dependence on previous values and thus less smoothing [S1, unofficial].
  - Typical value: $\alpha = 0.8$ is used for general methods and control signal filtering [S1, unofficial][S2].
  - Specific case: $\alpha = 0.3$ is used for IDR (Incremental Distance Representation) due to higher variance between plot points [S1, unofficial].

**Edge Cases and Pitfalls**
- **Initialization**: The value of $\hat{v}_0$ affects the initial transient. Common practice is to set $\hat{v}_0 = v_1$ to avoid bias [G].
- **Precision**: For very long sequences, the influence of $\hat{v}_0$ decays exponentially. If $\alpha$ is small, the effective memory is long, requiring careful handling of initial conditions.
- **Boundary**: The formula is undefined for $i=1$ without an initial condition.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exponential_moving_average_smoothing(v, alpha):
    """
    Applies Exponential Moving Average smoothing to a sequence of values.
    
    Args:
        v (list of float): Input sequence of raw values.
        alpha (float): Smoothing parameter, 0 < alpha <= 1.
        
    Returns:
        list of float: Smoothed sequence.
    """
    if not v:
        return []
    
    n = len(v)
    v_hat = [0.0] * n
    
    # Initialize with the first value to avoid bias from arbitrary zero [G]
    v_hat[0] = v[0]
    
    for i in range(1, n):
        v_hat[i] = alpha * v[i] + (1 - alpha) * v_hat[i-1]
        
    return v_hat
```

## Subtopics

### Exponential Decay Blur Filter
The parent method that applies this smoothing along the outermost two dimensions of a tensor. It uses the same recursive principle but in a spatial domain for image-order rendering. See [Exponential Decay Blur Filter](11279-exponential-decay-blur-filter.md).

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of samples. Each sample requires one multiplication and one addition.
- **Memory Complexity**: $O(1)$ auxiliary space if computed in-place or streaming, or $O(N)$ to store the output sequence.
- **Strengths**: Computationally efficient; no need for large kernels; adapts to local variance implicitly through the recursive structure.
- **Limitations**: Introduces phase lag; sensitive to initialization; not a linear phase filter.
- **When to use**: When low-latency smoothing is required, or when the noise characteristics are stationary and the trade-off between responsiveness and noise reduction is controlled by $\alpha$.
- **When not to use**: When strict linear phase is required, or when the noise has non-stationary characteristics that require adaptive $\alpha$.

## Background
The method originates from statistical smoothing techniques and has been applied in various domains including control systems and neural network training. In the context of neural radiance fields and volume rendering, it has been used to smooth control signals for robot actuation [S2] and to reduce variance in training metrics for neural lumigraph representations [S1, unofficial]. The specific parameter values ($\alpha=0.8$ and $\alpha=0.3$) are empirically derived from these applications.

## Sources
[S1] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
[S2] NeRF2Real: Sim2real Transfer of Vision-guided Bipedal Motion Skills using Neural Radiance Fields, https://doi.org/10.1109/icra48891.2023.10161544, fetched 2026-10-29
