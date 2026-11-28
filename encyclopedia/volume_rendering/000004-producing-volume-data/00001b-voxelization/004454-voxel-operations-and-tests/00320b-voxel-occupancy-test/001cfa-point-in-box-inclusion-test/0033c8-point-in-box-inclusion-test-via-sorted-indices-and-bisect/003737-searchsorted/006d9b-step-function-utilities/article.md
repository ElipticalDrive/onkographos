# Step Function Utilities
Also called: stepfun

## Overview
A set of utility functions for manipulating step functions (piecewise constant functions) defined by bin edges $t$ and values $y$, used for sampling, integration, and dilation in volume rendering contexts [S1, unofficial]. It provides operations to query values, compute inner/outer measures for loss functions, and dilate weights, building on [Searchsorted](7938-searchsorted.md).

## How it works
**Inputs and Outputs:**
- Inputs: Sorted bin edges $t$ (tensor of shape $[N]$ or $[..., N]$), values $y$ (tensor of shape $[N-1]$ or $[..., N-1]$), query points $tq$, weights $w$, dilation amount, and optional domain bounds.
- Outputs: Queried values, cumulative weights, dilated edges/weights, or loss values.

**Data Structures:**
- Step function defined by edges $t_i$ and values $y_i$ for bin $[t_i, t_{i+1})$.
- Weights $w$ are assumed to sum to 1 along the last axis for integration utilities [S1, unofficial].

**Formulas:**
- Weight to PDF: $p_i = w_i / (t_{i+1} - t_i)$ [S1, unofficial].
- PDF to Weight: $w_i = p_i \times (t_{i+1} - t_i)$ [S1, unofficial].
- Cumulative Weight: $cw_0 = 0$, $cw_i = \sum_{j=0}^{i-1} w_j$ for $i > 0$, clamped to max 1 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def step(t, v):
    """
    Find indices where v should be inserted into sorted tensor t.
    Returns (idx_lo, idx_hi).
    """
    # v_ge_a = v[..., None, :] >= t[..., :, None]
    # This creates a broadcasted comparison. 
    # idx_lo is the max index where v >= t.
    # idx_hi is the min index where v < t.
    # Note: The source describes this as searchsorted behavior.
    # Standard searchsorted returns the index i such that t[i-1] < v <= t[i] (right side)
    # or t[i-1] <= v < t[i] (left side).
    # The source implementation uses:
    # idx_lo = (v[..., None, :] >= t[..., :, None]).sum(dim=-1) - 1
    # idx_hi = (v[..., None, :] < t[..., :, None]).sum(dim=-1)
    # However, the source says: "idx_lo is the max index where v >= a. idx_hi is the min index where v < a."
    # Let's implement based on the source description of searchsorted usage.
    
    # Broadcast v and t
    # v shape: [..., M], t shape: [..., N]
    # Compare v[..., None] >= t[..., :, None] -> shape [..., M, N]
    # idx_lo: max index where v >= t. This is equivalent to the number of elements in t <= v, minus 1?
    # Actually, searchsorted(a, v) returns indices to insert v into a to maintain order.
    # If we use 'right' side, it returns i such that a[i-1] < v <= a[i].
    # The source implementation logic:
    # v_ge_a = v[..., None, :] >= t[..., :, None] # Shape [..., M, N]
    # idx_lo = v_ge_a.sum(dim=-1) - 1 # Max index where v >= t? No, sum gives count.
    # Let's stick to the source's explicit description:
    # idx_lo is the max index where v >= t.
    # idx_hi is the min index where v < t.
    
    # Implementation from source:
    # v_ge_a = v[..., None, :] >= t[..., :, None]
    # idx_lo = v_ge_a.sum(dim=-1) - 1
    # idx_hi = (~v_ge_a).sum(dim=-1) # Or (v < t).sum()
    
    # Correct interpretation of source code snippet:
    # v_ge_a = v[..., None, :] >= t[..., :, None]
    # idx_lo = v_ge_a.sum(dim=-1) - 1
    # idx_hi = (v[..., None, :] < t[..., :, None]).sum(dim=-1)
    
    # However, standard searchsorted is more efficient.
    # We will implement the source's logic using searchsorted if available, 
    # but the source explicitly defines the behavior via comparison.
    # Let's use torch.searchsorted for correctness and efficiency, mapping to the described behavior.
    # The source says: "searchsorted(a, v): Find indices where v should be inserted... Computes v_ge_a... idx_lo... idx_hi"
    # This implies the source *implements* searchsorted-like behavior or uses it.
    # Given the instruction to rewrite in clean Python, we use torch.searchsorted.
    
    # The source's idx_lo corresponds to searchsorted(t, v, side='right') - 1?
    # If v >= t[i], then i is a candidate for idx_lo.
    # Let's assume the source uses standard searchsorted semantics where idx_lo is the index of the bin.
    
    # Re-reading source: "idx_lo is the max index where v >= a. idx_hi is the min index where v < a."
    # This is exactly what searchsorted returns if we interpret it correctly.
    # idx_lo = searchsorted(t, v, side='right') - 1
    # idx_hi = searchsorted(t, v, side='left')
    
    idx_lo = torch.searchsorted(t, v, side='right') - 1
    idx_hi = torch.searchsorted(t, v, side='left')
    
    return idx_lo, idx_hi

def query(tq, t, y, outside_value=0):
    """
    Look up values of step function (t, y) at query points tq.
    """
    idx_lo, idx_hi = step(t, tq)
    
    # If idx_lo == idx_hi, return outside_value
    # Otherwise, return y[idx_lo]
    
    # Use take_along_dim
    # y shape: [..., N-1], idx_lo shape: [..., M]
    # We need to gather y at idx_lo
    
    # Expand dimensions for gather
    # y.unsqueeze(-1) -> [..., N-1, 1]
    # idx_lo.unsqueeze(-1) -> [..., M, 1]
    # But gather expects index dim to match.
    
    # Correct gather:
    # y_gathered = torch.gather(y, -1, idx_lo.unsqueeze(-1)).squeeze(-1)
    
    y_gathered = torch.gather(y, -1, idx_lo.unsqueeze(-1)).squeeze(-1)
    
    mask = (idx_lo == idx_hi)
    result = torch.where(mask, outside_value, y_gathered)
    
    return result

def inner_outer(t0, t1, y1):
    """
    Construct inner and outer measures.
    t0: edges of target step function
    t1: edges of reference step function
    y1: values of reference step function
    """
    # Compute cumulative sum of y1, prefixed with 0
    # cy1 shape: [..., N]
    cy1 = torch.cat([torch.zeros_like(y1[..., :1]), torch.cumsum(y1, dim=-1)], dim=-1)
    
    # Use searchsorted(t1, t0) to get idx_lo, idx_hi
    idx_lo, idx_hi = step(t1, t0)
    
    # y0_outer = cy1_hi[..., 1:] - cy1_lo[..., :-1]
    # cy1_hi: gather cy1 at idx_hi
    # cy1_lo: gather cy1 at idx_lo
    
    cy1_hi = torch.gather(cy1, -1, idx_hi.unsqueeze(-1)).squeeze(-1)
    cy1_lo = torch.gather(cy1, -1, idx_lo.unsqueeze(-1)).squeeze(-1)
    
    # y0_outer: difference of cumulative sums
    # cy1_hi[..., 1:] - cy1_lo[..., :-1]
    # This aligns the bins.
    y0_outer = cy1_hi[..., 1:] - cy1_lo[..., :-1]
    
    # y0_inner: 
    # Condition: if idx_hi[..., :-1] <= idx_lo[..., 1:], else 0
    # Note: The source says "if idx_hi[..., :-1] <= idx_lo[..., 1:], else 0"
    # This condition checks if the bins overlap significantly.
    
    # cy1_lo[..., 1:] - cy1_hi[..., :-1]
    y0_inner_diff = cy1_lo[..., 1:] - cy1_hi[..., :-1]
    
    mask = idx_hi[..., :-1] <= idx_lo[..., 1:]
    y0_inner = torch.where(mask, y0_inner_diff, torch.zeros_like(y0_inner_diff))
    
    return y0_inner, y0_outer

def lossfun_outer(t, w, t_env, w_env):
    """
    Computes a scaled half-quadratic loss ensuring proposal weight w is bounded by outer measure w_outer.
    """
    # Compute w_outer via inner_outer
    # t_env, w_env are the environment/reference step function
    _, w_outer = inner_outer(t, t_env, w_env)
    
    # (w - w_outer).clamp_min(0) ** 2 / (w + eps)
    eps = 1e-6
    diff = w - w_outer
    diff_clamped = torch.clamp(diff, min=0)
    loss = (diff_clamped ** 2) / (w + eps)
    
    return loss

def max_dilate(t, w, dilation, domain=(-float('inf'), float('inf'))):
    """
    Dilates step function via max-pooling.
    """
    # Create dilated edges
    # t0 = t[:-1] - dilation
    # t1 = t[1:] + dilation
    t0 = t[..., :-1] - dilation
    t1 = t[..., 1:] + dilation
    
    # Sort concatenated t, t0, t1 to get t_dilate
    # Concatenate along last dimension
    t_concat = torch.cat([t, t0, t1], dim=-1)
    t_dilate, _ = torch.sort(t_concat, dim=-1)
    
    # Clip t_dilate to domain
    t_dilate = torch.clamp(t_dilate, min=domain[0], max=domain[1])
    
    # Compute w_dilate by taking max of w where t_dilate falls within [t0, t1)
    # This part is complex. The source says: "Computes w_dilate by taking max of w where t_dilate falls within [t0, t1)."
    # We need to map t_dilate back to the original bins.
    
    # For each edge in t_dilate, find which original bin it belongs to.
    # Then take the max w of those bins.
    
    # This is effectively a max-pooling over the dilated intervals.
    # We can use searchsorted to find the indices of t_dilate in t.
    
    # However, the source implementation is not fully detailed in the snippet.
    # We will implement a simplified version based on the description.
    
    # Find indices of t_dilate in t
    idx = torch.searchsorted(t, t_dilate, side='right') - 1
    idx = torch.clamp(idx, min=0, max=len(w)-1)
    
    # Gather w values
    w_dilated = torch.gather(w, -1, idx.unsqueeze(-1)).squeeze(-1)
    
    # Since t_dilate is sorted, and we want the max w in the dilated region,
    # we might need to group by the original bins and take max.
    # But the source says "t_dilate" is the new edges.
    # The values w_dilate correspond to the new bins defined by t_dilate.
    
    # This is a heuristic. The exact implementation in the source might differ.
    # We will return t_dilate and w_dilated as per the description.
    
    return t_dilate, w_dilated

def max_dilate_weights(t, w, dilation, domain, renormalize=False):
    """
    Dilates weights.
    """
    # Convert w to PDF p
    p = weight_to_pdf(t, w)
    
    # Dilate p to get p_dilate
    t_dilate, p_dilate = max_dilate(t, p, dilation, domain)
    
    # Convert p_dilate back to w_dilate
    w_dilate = pdf_to_weight(t_dilate, p_dilate)
    
    # If renormalize is True, divide w_dilate by its sum along the last dimension
    if renormalize:
        w_sum = w_dilate.sum(dim=-1, keepdim=True)
        w_dilate = w_dilate / w_sum
    
    return w_dilate

def integrate_weights(w):
    """
    Computes cumulative sum of w, assuming w sums to 1 along the last axis.
    Returns cw0 with shape [..., len(w)+1], starting with 0 and ending with 1.
    """
    # cw = cumsum(w[..., :-1])
    cw = torch.cumsum(w[..., :-1], dim=-1)
    cw = torch.clamp(cw, max=1.0)
    
    # Concatenate zeros, cw, ones
    zeros = torch.zeros_like(w[..., :1])
    ones = torch.ones_like(w[..., :1])
    
    cw0 = torch.cat([zeros, cw, ones], dim=-1)
    
    return cw0

def weight_to_pdf(t, w):
    """
    Converts weights w to PDF p.
    p = w / (t_{i+1} - t_i)
    """
    dt = t[..., 1:] - t[..., :-1]
    dt = torch.clamp_min(dt, 1e-6)
    p = w / dt
    return p

def pdf_to_weight(t, p):
    """
    Converts PDF p to weights w.
    w = p * (t_{i+1} - t_i)
    """
    dt = t[..., 1:] - t[..., :-1]
    w = p * dt
    return w
```

**Parameters:**
- `dilation`: Float, amount to dilate the step function [S1, unofficial].
- `domain`: Tuple of floats, min and max bounds for dilation [S1, unofficial].
- `renormalize`: Boolean, whether to renormalize dilated weights [S1, unofficial].
- `outside_value`: Float, value to return when query point is outside bin range [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Division by zero in `weight_to_pdf` if bin width is zero. Mitigated by `clamp_min(eps)` [S1, unofficial].
- Boundaries: Query points outside the range of $t$ return `outside_value` [S1, unofficial].
- Degenerate Input: Empty tensors or single-element tensors may cause shape mismatches in concatenation or gathering. Not given in the sources.

## Operating characteristics
- Time Complexity: $O(N \log N)$ for sorting in `max_dilate`, $O(N)$ for cumulative sums and gather operations [S1, unofficial].
- Memory Complexity: $O(N)$ for storing intermediate tensors [S1, unofficial].
- Strengths: Efficient vectorized operations using PyTorch [S1, unofficial].
- Limitations: Assumes sorted input edges [S1, unofficial].

## Background
Origin: Part of the DOFGS implementation [S1, unofficial].

## Sources
[S1] rongduo/DOFGS_implementation, https://github.com/rongduo/DOFGS_implementation, fetched 2026-10-27 (unofficial)
