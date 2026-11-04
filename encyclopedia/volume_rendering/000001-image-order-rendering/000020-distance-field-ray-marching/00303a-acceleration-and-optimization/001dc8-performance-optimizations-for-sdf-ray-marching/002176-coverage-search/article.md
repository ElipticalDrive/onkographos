# Coverage Search

Also called: SDF Raymarching Algorithm: Coverage Search

## Overview
Coverage Search is a distance-field ray marching algorithm designed to determine ray hit or miss status in fewer iterations than standard sphere tracing by using a stack to manage coverage spans along the ray [S1, unofficial]. It sets itself apart from siblings like [Aggressive Ray Marching with SDF](2326-aggressive-ray-marching-with-sdf.md) by explicitly tracking intervals of guaranteed empty or occupied space, allowing it to skip large portions of the ray where the signed distance function (SDF) would otherwise be evaluated repeatedly without progress.

## How it works
**Inputs and Outputs:**
*   `RayOrigin`: The starting position of the ray.
*   `RayDirection`: The normalized direction vector of the ray.
*   `TravelStart`: Estimated start travel distance (e.g., from bounding volume intersection).
*   `TravelEnd`: Estimated end travel distance (e.g., from bounding volume intersection).
*   `SignedDistanceFunction(p)`: A function returning the signed distance from point `p` to the nearest surface.
*   `Position` (output): The 3D position of the hit or the end of the ray.
*   Return value: `true` if a hit occurred, `false` otherwise.

**Data Structures:**
*   `Coverage`: A struct containing:
    *   `Low`: Float, the start of the coverage span along the ray parameter.
    *   `High`: Float, the end of the coverage span along the ray parameter.
    *   `Sign`: Integer, `1` if the span is inside the object (negative distance), `-1` if outside (positive distance), `0` for neutral/initial states.
*   `Stack`: A dynamic array or fixed-size buffer of `Coverage` structs. The source notes a required size of approximately 10 for GLSL implementations [S1, unofficial].

**Algorithm Steps:**
1.  Initialize the stack.
2.  Calculate the initial pivot point in the middle of the travel bounds.
3.  Evaluate the SDF at the pivot.
4.  Determine the coverage span of the pivot based on the SDF value.
5.  Iterate through the stack, merging or subdividing spans until the ray start is subsumed by a known coverage span.
6.  Determine the final hit position based on the sign of the final cursor span.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CoverageSearch(RayOrigin, RayDirection, TravelStart, TravelEnd):
    # 1. Initialize Stack
    # The stack manages spans of travel distance.
    # Initial push: a neutral span at the end of the ray.
    Stack = []
    Stack.append(Coverage(High=TravelEnd, Low=TravelEnd, Sign=0))
    
    # 2. Initial Pivot Calculation
    PivotTravel = (TravelStart + TravelEnd) * 0.5
    PivotPoint = RayOrigin + RayDirection * PivotTravel
    PivotRadius = SignedDistanceFunction(PivotPoint)
    
    # 3. Push Initial Coverage Span
    # The span extends from PivotTravel minus the distance to PivotTravel plus the distance.
    # The sign indicates if the pivot is inside (-1) or outside (1) the object.
    # Note: sign() returns -1 for negative, 1 for positive, 0 for zero.
    InitialSign = 1 if PivotRadius > 0 else (-1 if PivotRadius < 0 else 0)
    SpanLow = PivotTravel - abs(PivotRadius)
    SpanHigh = PivotTravel + abs(PivotRadius)
    Stack.append(Coverage(High=SpanHigh, Low=SpanLow, Sign=InitialSign))
    
    # 4. Initialize Cursor
    # The cursor tracks the current "frontier" of known coverage starting from TravelStart.
    Cursor = Coverage(High=TravelStart, Low=TravelStart, Sign=0)
    
    # 5. Main Loop
    while len(Stack) > 0:
        Top = Stack[-1]
        
        # Check if the top of the stack overlaps with the current cursor's known high point.
        # If Stack[Top].Low <= Cursor.High, the stack span is reachable/merged.
        if Top.Low <= Cursor.High:
            # Merge the stack span into the cursor.
            # The cursor expands to cover the stack span.
            Cursor = Stack.pop()
        else:
            # The stack span is ahead of the cursor. We need to subdivide.
            # Calculate a new pivot between the cursor's high and the stack's low.
            PivotTravel = (Top.Low + Cursor.High) * 0.5
            PivotPoint = RayOrigin + RayDirection * PivotTravel
            PivotRadius = SignedDistanceFunction(PivotPoint)
            
            # Determine the new coverage span for this pivot.
            NextSign = 1 if PivotRadius > 0 else (-1 if PivotRadius < 0 else 0)
            NextLow = PivotTravel - abs(PivotRadius)
            NextHigh = PivotTravel + abs(PivotRadius)
            Next = Coverage(High=NextHigh, Low=NextLow, Sign=NextSign)
            
            # Check for merging conditions.
            # If the new span has a different sign than the stack top AND overlaps with it,
            # we can potentially merge or adjust the stack top's low bound.
            # The condition abs(Stack[Top].Sign + Next.Sign) > 0 implies signs are not opposite (i.e., same sign or one is zero).
            # However, the source logic specifically checks if they can be merged/adjusted.
            if abs(Top.Sign + Next.Sign) > 0 and Top.Low <= Next.High:
                # Adjust the stack top's low bound to the new span's low.
                # This effectively merges the new span into the existing stack top if they are compatible.
                Top.Low = Next.Low
            else:
                # Push the new span onto the stack.
                Stack.append(Next)
    
    # 6. Determine Result
    # If the final cursor sign is -1 (inside), we hit the object.
    # The hit position is at the start of the inside span (Cursor.Low).
    # If the cursor sign is not -1, we missed.
    if Cursor.Sign == -1:
        # Clamp to TravelStart to ensure we don't go before the start bound.
        HitTravel = max(TravelStart, Cursor.Low)
        Position = RayOrigin + RayDirection * HitTravel
        return True, Position
    else:
        Position = RayOrigin + RayDirection * TravelEnd
        return False, Position

class Coverage:
    def __init__(self, High, Low, Sign):
        self.High = High
        self.Low = Low
        self.Sign = Sign
```

**Parameters:**
*   `TravelStart`: Typically derived from the intersection of the ray with a simple bounding volume (e.g., AABB or sphere) [S1, unofficial].
*   `TravelEnd`: Typically derived from the intersection of the ray with a simple bounding volume [S1, unofficial].
*   `Stack Size`: Experimentally, a stack size of approximately 10 was required for the GLSL implementation to work correctly, whereas an initial non-shader implementation required only a size of 4 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Shader Occupancy:** The stack consumes shader registers/occupancy, which can reduce performance on hardware with limited resources [S1, unofficial].
*   **Branching:** The algorithm is branchier than standard sphere tracing, which may impact performance on GPUs [S1, unofficial].
*   **SDF Cost:** The approach is only worthwhile if the `SignedDistanceFunction` is sufficiently expensive to justify the overhead of stack management and branching [S1, unofficial].
*   **Discrepancy in Stack Size:** There is an unexplained discrepancy between the required stack size in the GLSL implementation (10) and the initial analysis implementation (4), potentially due to a bug in the GLSL version [S1, unofficial].
*   **Precision:** The calculation of `SpanLow` and `SpanHigh` uses `abs(PivotRadius)`. If `PivotRadius` is very small, precision issues may arise, though not explicitly detailed in the source [G].

## Operating characteristics
*   **Performance:** In the ideal case, Coverage Search performs approximately the same number of `SignedDistanceFunction` calls as sphere tracing. However, it results in half as many calls when the average signed distance approaches zero without hitting zero [S1, unofficial].
*   **Suitability:** The method is best suited for environments suitable for recursive algorithms and where the SDF is expensive to evaluate [S1, unofficial]. It is less suitable for scenes with simple, cheap SDFs or hardware with strict occupancy limits.
*   **Comparison to Siblings:** Unlike [Aggressive Ray Marching with SDF](2326-aggressive-ray-marching-with-sdf.md) which may skip steps based on distance thresholds, Coverage Search explicitly manages intervals. It differs from [SDF-Accelerated Volumetric Ray Marching](3681-sdf-accelerated-volumetric-ray-marching.md) by focusing on hit/miss determination via span coverage rather than volumetric accumulation.

## Background
The Coverage Search algorithm was developed by the author of the source material as an effort to solve ray hit/miss determination in fewer iterations than standard sphere tracing [S1, unofficial]. It was designed to address cases where standard raymarching requires substantially more iterations, such as hitting an object dead-on, hitting at an acute angle, or running parallel to an object without hitting it [S1, unofficial]. The algorithm was initially analyzed in a non-shader implementation and later ported to GLSL [S1, unofficial].

## Sources
[S1] SDF Raymarching, https://zone.dog/braindump/sdf_marching/, fetched 2026-10-29 (unofficial)
