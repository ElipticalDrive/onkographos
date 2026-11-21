# Coverage Search

Also called: SDF Raymarching Algorithm: Coverage Search

## Overview
Coverage Search is a distance-field ray marching algorithm designed to determine ray hit or miss status in fewer iterations than standard sphere tracing by using a stack to manage coverage spans along the ray [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   `RayOrigin`: The starting position of the ray [S1, unofficial].
*   `RayDirection`: The normalized direction vector of the ray [S1, unofficial].
*   `TravelStart`: Estimated start travel distance, typically from ray intersection with a simple bounding volume [S1, unofficial].
*   `TravelEnd`: Estimated end travel distance, typically from ray intersection with a simple bounding volume [S1, unofficial].
*   `SignedDistanceFunction(p)`: A scene-dependent function returning the signed distance from point `p` to the nearest surface [S1, unofficial].
*   `Position` (output): The 3D position of the hit or the end of the ray [S1, unofficial].
*   Return value: `true` if a hit occurred, `false` otherwise [S1, unofficial].

**Data Structures:**
*   `Coverage`: A struct containing:
    *   `Low`: Float, the start of the coverage span along the ray parameter [S1, unofficial].
    *   `High`: Float, the end of the coverage span along the ray parameter [S1, unofficial].
    *   `Sign`: Integer, `1` if the span is outside (positive distance), `-1` if inside (negative distance), `0` for neutral/initial states [S1, unofficial].
*   `Stack`: A dynamic array or fixed-size buffer of `Coverage` structs. The source notes a required size of approximately 10 for GLSL implementations [S1, unofficial].

**Algorithm Steps:**
1.  Initialize the stack with a neutral span at `TravelEnd` and an initial pivot span [S1, unofficial].
2.  Calculate the initial pivot point in the middle of the travel bounds [S1, unofficial].
3.  Evaluate the SDF at the pivot and determine its coverage span [S1, unofficial].
4.  Iterate through the stack, merging or subdividing spans until the ray start is subsumed by a known coverage span [S1, unofficial].
5.  Determine the final hit position based on the sign of the final cursor span [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def SignedDistanceFunction(p):
    # Scene-dependent SDF. Defined here as a unit sphere at origin for completeness.
    return math.sqrt(p[0]**2 + p[1]**2 + p[2]**2) - 1.0

def sign(x):
    if x > 0: return 1
    elif x < 0: return -1
    else: return 0

class Coverage:
    def __init__(self, Low, High, Sign):
        self.Low = Low
        self.High = High
        self.Sign = Sign

def CoverageSearch(RayOrigin, RayDirection, TravelStart, TravelEnd):
    Stack = []
    # Push neutral span at TravelEnd
    Stack.append(Coverage(Low=TravelEnd, High=TravelEnd, Sign=0))
    
    # Initial pivot calculation
    PivotTravel = (TravelStart + TravelEnd) * 0.5
    PivotPoint = [RayOrigin[i] + RayDirection[i] * PivotTravel for i in range(3)]
    PivotRadius = SignedDistanceFunction(PivotPoint)
    
    # Push initial coverage span
    SpanLow = PivotTravel - abs(PivotRadius)
    SpanHigh = PivotTravel + abs(PivotRadius)
    InitialSign = sign(PivotRadius)
    Stack.append(Coverage(Low=SpanLow, High=SpanHigh, Sign=InitialSign))
    
    # Initialize Cursor
    Cursor = Coverage(Low=TravelStart, High=TravelStart, Sign=0)
    
    while len(Stack) > 0:
        Top = Stack[-1]
        
        # Check if stack span overlaps with cursor's known high point
        if Top.Low <= Cursor.High:
            popped = Stack.pop()
            # Merge popped span into cursor
            if popped.High > Cursor.High:
                Cursor.High = popped.High
            if popped.Sign == -1:
                Cursor.Sign = -1
        else:
            # Subdivide between cursor high and stack low
            PivotTravel = (Top.Low + Cursor.High) * 0.5
            PivotPoint = [RayOrigin[i] + RayDirection[i] * PivotTravel for i in range(3)]
            PivotRadius = SignedDistanceFunction(PivotPoint)
            
            NextSign = sign(PivotRadius)
            NextLow = PivotTravel - abs(PivotRadius)
            NextHigh = PivotTravel + abs(PivotRadius)
            Next = Coverage(Low=NextLow, High=NextHigh, Sign=NextSign)
            
            # Check merge condition: signs are not opposite (abs(sum) > 0) and spans overlap
            if abs(Top.Sign + Next.Sign) > 0 and Top.Low <= Next.High:
                Top.Low = Next.Low
            else:
                Stack.append(Next)
                
    # Determine result
    if Cursor.Sign == -1:
        HitTravel = max(TravelStart, Cursor.Low)
        Position = [RayOrigin[i] + RayDirection[i] * HitTravel for i in range(3)]
        return True, Position
    else:
        Position = [RayOrigin[i] + RayDirection[i] * TravelEnd for i in range(3)]
        return False, Position
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

## Operating characteristics
*   **Performance:** In the ideal case, Coverage Search performs approximately the same number of `SignedDistanceFunction` calls as sphere tracing. However, it results in half as many calls when the average signed distance approaches zero without hitting zero [S1, unofficial].
*   **Suitability:** The method is best suited for environments suitable for recursive algorithms and where the SDF is expensive to evaluate [S1, unofficial]. It is less suitable for scenes with simple, cheap SDFs or hardware with strict occupancy limits [S1, unofficial].

## Background
The Coverage Search algorithm was developed by the author of the source material as an effort to solve ray hit/miss determination in fewer iterations than standard sphere tracing [S1, unofficial]. It was designed to address cases where standard raymarching requires substantially more iterations, such as hitting an object dead-on, hitting at an acute angle, or running parallel to an object without hitting it [S1, unofficial]. The algorithm was initially analyzed in a non-shader implementation and later ported to GLSL [S1, unofficial].

## Sources
[S1] SDF Raymarching, https://zone.dog/braindump/sdf_marching/, fetched 2026-10-29 (unofficial)
