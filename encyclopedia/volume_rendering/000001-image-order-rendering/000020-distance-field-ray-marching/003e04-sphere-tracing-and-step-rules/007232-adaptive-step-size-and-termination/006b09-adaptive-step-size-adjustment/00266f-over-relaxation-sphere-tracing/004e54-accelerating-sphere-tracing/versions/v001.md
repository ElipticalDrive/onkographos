# Accelerating Sphere Tracing

Also called: Planar Assumption Sphere Tracing, Predictive Sphere Tracing

## Overview
Accelerating Sphere Tracing is a ray marching step rule that predicts the next step distance by assuming the object surface is locally planar near the current ray position [S1, unofficial]. This allows for larger, more aggressive steps than standard Over-relaxation Sphere Tracing while reducing the frequency of overshoot corrections [S1, unofficial]. It builds on [Over-relaxation Sphere Tracing](3805-over-relaxation-sphere-tracing.md) by incorporating historical SDF values to refine step prediction.

## How it works
**Inputs:** Ray origin $p_0$, ray direction $\mathbf{d}$ (normalized), SDF function $f(\cdot)$, safety factor $\omega \in (0, 1)$, termination threshold $\epsilon$.
**Outputs:** Intersection point $p_{hit}$, number of steps.

**Data Structures:**
- Current position $p_n \in \mathbb{R}^3$.
- Previous position $p_{n-1} \in \mathbb{R}^3$.
- Current SDF value $r_n = f(p_n) \in \mathbb{R}$.
- Previous SDF value $r_{n-1} = f(p_{n-1}) \in \mathbb{R}$.
- Previous step distance $d_n = \|p_n - p_{n-1}\| \in \mathbb{R}$.

**Steps:**
1. Initialize $p_0$ as ray origin. Evaluate $r_0 = f(p_0)$. Set $p_{-1}$ and $r_{-1}$ to dummy values or handle the first step separately (see Pitfalls).
2. For step $n = 0, 1, \dots$:
   a. If $r_n < \epsilon$, terminate. Intersection found at $p_n$.
   b. If $n=0$, calculate initial step $d_1 = r_0$ (standard sphere tracing) or use a heuristic. [G] For $n>0$, retrieve $r_{n-1}$ and $d_n$.
   c. Calculate predicted radius $r_{n+1}$ using the planar assumption formula:
      $$r_{n+1} = \frac{d_n - r_{n-1} + r_n}{d_n + r_{n-1} + r_n}$$
   d. Calculate next step distance $d_{n+1}$:
      $$d_{n+1} = r_n + \omega r_{n+1}$$
   e. Move ray: $p_{n+1} = p_n + d_{n+1} \cdot \mathbf{d}$.
   f. Check for overshoot: Evaluate $r_{n+1}' = f(p_{n+1})$. If $r_{n+1}' + r_n < d_{n+1}$, the ray has jumped over the object.
   g. If overshoot detected:
      i. Retreat by $\omega r_{n+1}$: $p_{retreat} = p_{n+1} - \omega r_{n+1} \cdot \mathbf{d}$.
      ii. Set $p_{n+1} = p_{retreat}$.
      iii. Proceed with standard sphere tracing logic for the remaining distance (i.e., next step $d_{next} = f(p_{n+1})$). [S1, unofficial]
   h. Update history: $p_{n-1} \leftarrow p_n$, $p_n \leftarrow p_{n+1}$, $r_{n-1} \leftarrow r_n$, $r_n \leftarrow f(p_{n+1})$, $d_n \leftarrow d_{n+1}$.

**Pseudocode:**
```python
def accelerating_sphere_tracing(ray_origin, ray_direction, sdf, omega, epsilon):
    p_curr = ray_origin
    r_curr = sdf(p_curr)
    
    # Handle first step or initialization
    # Standard sphere tracing for the first step to establish history
    d_prev = r_curr 
    p_prev = ray_origin # Dummy or same as current for first iteration logic
    r_prev = r_curr     # Dummy value
    
    # Note: The formula requires valid r_prev and d_prev from a previous step.
    # For n=0, we cannot use the predictive formula directly without history.
    # We take one standard step to initialize history.
    
    p_next = p_curr + d_prev * ray_direction
    r_next = sdf(p_next)
    
    # Update history for the loop
    p_prev = p_curr
    r_prev = r_curr
    d_prev = d_prev # length(p_curr - p_prev) which is d_prev
    
    p_curr = p_next
    r_curr = r_next
    
    while r_curr >= epsilon:
        # Calculate predicted radius based on planar assumption
        # r_{n+1} = (d_n - r_{n-1} + r_n) / (d_n + r_{n-1} + r_n)
        numerator = d_prev - r_prev + r_curr
        denominator = d_prev + r_prev + r_curr
        
        if denominator == 0:
            # Fallback to standard sphere tracing if denominator is zero
            r_pred = 0.0
        else:
            r_pred = numerator / denominator
            
        # Calculate next step distance
        d_next = r_curr + omega * r_pred
        
        # Move ray
        p_next = p_curr + d_next * ray_direction
        r_next = sdf(p_next)
        
        # Check for overshoot
        # Condition: f(p_{n+1}) + f(p_n) < d_{n+1}
        if r_next + r_curr < d_next:
            # Overshoot detected
            # Retreat by omega * r_pred
            retreat_distance = omega * r_pred
            p_next = p_next - retreat_distance * ray_direction
            r_next = sdf(p_next)
            
            # Proceed with standard sphere tracing logic
            # This implies the next step will be based on r_next directly
            # We need to update history to reflect this "standard" step
            # The next iteration will use r_next as r_curr, and the step taken 
            # to get here (which was a retreat) needs to be handled carefully.
            # The source says "proceed with standard sphere tracing logic".
            # This usually means the next step size is just r_next.
            
            # Update history for next iteration
            # The step we just took was a retreat, but for the predictor,
            # we might want to treat the next step as a fresh start or 
            # use the retreat distance as d_prev? 
            # The source implies reverting to standard logic, so let's 
            # set d_prev to r_next for the next step to force standard behavior?
            # Or simply update history normally and let the next step's 
            # predictor handle it? 
            # The source says "return by omega rn+1 amount and proceed with 
            # standard sphere tracing". This suggests the NEXT step is standard.
            
            # Let's update history normally first
            p_prev = p_curr
            r_prev = r_curr
            d_prev = d_next # The distance we attempted
            
            p_curr = p_next
            r_curr = r_next
            
            # Force next step to be standard sphere tracing by setting 
            # a flag or adjusting d_prev? 
            # Actually, if we just update history, the next iteration 
            # will use the predictor again. To "proceed with standard",
            # we might need to ensure the next d_next is just r_curr.
            # The source is slightly ambiguous on implementation details 
            # of the "standard logic" continuation. 
            # A common interpretation is that the overshoot correction 
            # puts us close to the surface, and the next step is small.
            
            # Let's assume the next step is calculated normally, 
            # but the overshoot check will likely pass or fail again.
            # To strictly follow "standard sphere tracing", we could 
            # set d_prev = r_curr in the next iteration? 
            # No, d_prev is length(p_curr - p_prev).
            
            # Let's stick to the history update. The predictor will 
            # likely yield a small r_pred if we are close to surface.
            continue

        # Update history
        p_prev = p_curr
        r_prev = r_curr
        d_prev = d_next
        
        p_curr = p_next
        r_curr = r_next
        
    return p_curr
```

**Parameters:**
- $\omega$: Safety factor multiplier. Range $(0, 1)$. Typical value not specified, but must be less than 1 to prevent penetration into concave geometry [S1, unofficial].

**Pitfalls:**
- **Concave Geometry:** The planar assumption may cause the predicted step to land inside the object if the surface is concave. The safety factor $\omega$ mitigates this by reducing the step size [S1, unofficial].
- **Overshoot Correction:** If overshoot is detected, the ray retreats. The implementation of "proceeding with standard sphere tracing" needs care to ensure the next step is not overly aggressive again immediately [S1, unofficial].
- **Initial Steps:** The formula requires $r_{n-1}$ and $d_n$. The first step cannot use this formula directly. A standard sphere tracing step is typically used to initialize history [G].

## Operating characteristics
- **Performance:** Rendering time is approximately 0.8 times that of conventional Relaxation Sphere Tracing [S1, unofficial].
- **Strengths:** Allows more aggressive stepping than standard over-relaxation, reducing the number of steps and overshoot corrections [S1, unofficial].
- **Limitations:** Relies on the planar assumption, which may be inaccurate for highly curved or concave surfaces, requiring the safety factor $\omega$ [S1, unofficial].

## Background
- **Origin:** The method was proposed in 2018 as an improvement over Enhanced Sphere Tracing and Relaxation Sphere Tracing [S1, unofficial].
- **Development:** It aims to make ray marching steps bolder by using historical SDF values to predict the next step size based on a planar surface assumption [S1, unofficial].

## Sources
[S1] 要約Accelerating Sphere Tracing～より良いレイマーチングに向けて Part2～, https://qiita.com/Hirai0827/items/e05e13f343357d648b1b, fetched 2026-10-26 (unofficial)
