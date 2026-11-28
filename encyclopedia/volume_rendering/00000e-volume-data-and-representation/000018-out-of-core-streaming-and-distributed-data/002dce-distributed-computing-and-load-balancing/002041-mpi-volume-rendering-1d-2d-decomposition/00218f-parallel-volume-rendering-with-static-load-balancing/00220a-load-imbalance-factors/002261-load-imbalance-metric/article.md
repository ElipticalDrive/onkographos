# Load Imbalance Metric


## Overview
The Load Imbalance Metric quantifies the performance degradation caused by uneven workload distribution across processors in parallel volume ray-casting on unstructured grids [S1, unofficial][S2, unofficial]. It normalizes the disparity between the slowest and average processors into a dimensionless ratio of inefficiency [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: $t_{max}$, the maximum rendering time among all processors; $t_{avg}$, the average rendering time across all processors [S1, unofficial][S2, unofficial].
- **Output**: $l$, the load imbalance metric value, where $0 \le l < 1$ [S1, unofficial][S2, unofficial].

**Data Structures**
Not given in the sources.

**Formula**
The metric is calculated as:
$$ l = 1 - \frac{t_{avg}}{t_{max}} $$
where:
- $t_{avg}$ is the average rendering time across all processors [S1, unofficial][S2, unofficial].
- $t_{max}$ is the maximum rendering time among all processors [S1, unofficial][S2, unofficial].
- $P$ is the total number of processors, used to compute $t_{avg} = \frac{1}{P} \sum_{i=1}^{P} t_i$ and $t_{max} = \max_{i=1}^{P} t_i$ [G].

**Steps**
1. Measure the total rendering time $t_i$ for each processor $i$ from $1$ to $P$ [G].
2. Compute the average time $t_{avg}$ by summing all $t_i$ and dividing by $P$ [G].
3. Identify the maximum time $t_{max}$ from the set of all $t_i$ [G].
4. Calculate the ratio $t_{avg} / t_{max}$ and subtract it from $1$ to obtain $l$ [S1, unofficial][S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_load_imbalance(processor_times):
    """
    Calculate the load imbalance metric for a set of processor rendering times.
    
    Args:
        processor_times (list of float): List of total rendering times for each processor.
        
    Returns:
        float: The load imbalance metric l.
    """
    if not processor_times:
        return 0.0
    
    t_avg = sum(processor_times) / len(processor_times)
    t_max = max(processor_times)
    
    if t_max == 0:
        return 0.0
        
    return 1.0 - (t_avg / t_max)
```

**Edge Cases and Pitfalls**
- **Distribution Blindness**: The formulation does not reveal the actual distribution of the imbalanced load, only the aggregate degradation [S1, unofficial][S2, unofficial].
- **Zero Time**: If rendering time is zero, the metric is undefined; a guard returning $0.0$ is required [G].

## Operating characteristics
**Time and Memory Complexity**
- **Time**: $O(P)$ to compute $t_{avg}$ and $t_{max}$ from $P$ processor times [G].
- **Memory**: $O(1)$ auxiliary space if times are streamed, or $O(P)$ to store the list of times [G].

**Measured Performance**
In tests on flow data with a 480x480 image size using 32 processors, load imbalance degraded overall performance by at least 20% [S1, unofficial][S2, unofficial]. When using a zoom-in view, the difference between maximum and minimum time increased from about 40% to 66%, causing the proportion of time devoted to ray casting due to load imbalance to rise to approximately 33% [S1, unofficial][S2, unofficial].

**Strengths and Limitations**
- **Strengths**: Simple to compute from standard timing logs [G].
- **Limitations**: Does not indicate which processors are imbalanced or the nature of the imbalance [G]. It only provides an aggregate measure of degradation [S1, unofficial][S2, unofficial].

**When to Use**
Not given in the sources.

## Background
The metric was developed to analyze parallel volume ray-casting for unstructured-grid data on distributed-memory architectures [S1, unofficial][S2, unofficial]. The sources note five primary factors contributing to load imbalance, including subvolume size, though the complete list is truncated in the provided excerpts [S1, unofficial][S2, unofficial]. It has been used to evaluate optimization techniques such as ray-segment merging in unstructured grid rendering [S1, unofficial][S2, unofficial].

## Sources
[S1] DTIC ADA301855: Parallel Volume Ray-Casting for Unstructured-Grid Data on Distributed-Memory Architectures., https://archive.org/details/DTIC_ADA301855, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19960003512: Parallel volume ray-casting for unstructured-grid data on distributed-memory architectures, https://archive.org/details/NASA_NTRS_Archive_19960003512, fetched 2026-10-26 (unofficial)
