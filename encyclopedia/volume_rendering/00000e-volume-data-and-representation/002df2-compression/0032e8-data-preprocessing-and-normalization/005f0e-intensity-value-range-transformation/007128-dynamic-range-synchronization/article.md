# Dynamic Range Synchronization

Also called: Dynamic Range Synchronization via Hypertools, VISU range broadcast, inter-tool dynamic range normalization

## Overview
Dynamic Range Synchronization ensures consistent normalization of floating-point images or volume slices across multiple independent visualization instances [S1, unofficial]. It solves the problem of disparate intensity thresholds in peer-to-peer or master/slave visualization setups, where different tools may display the same data with different scaling, making pixel-by-pixel comparison difficult [S1, unofficial]. Unlike [Volume Normalization](1823-volume-normalization.md) which typically normalizes a single dataset to a standard range, this method synchronizes the *display parameters* of multiple existing instances to a globally computed bound [S1, unofficial].

## How it works
**Inputs:** A list of active visualization interpreters (widgets), a reference to the current image/volume data object (`curr-img`), and the ability to query min/max values and set dynamic ranges on these interpreters.
**Outputs:** Updated dynamic range settings on all synchronized visualization instances.

**Data Structures:**
*   `visu-list`: A list of interpreter identifiers (e.g., Tcl variable names or object handles) corresponding to active visualization windows.
*   `min`, `max`: Lists of scalar values representing the current dynamic range bounds reported by each instance.
*   `fmin`, `fmax`: Scalar values representing the computed global minimum and maximum intensity values.

**Steps:**
1.  **Initialization:** Retrieve the list of all active interpreters. Initialize `visu-list`, `min`, and `max` as empty lists.
2.  **Filtering:** Iterate through the active interpreters. Identify those belonging to the visualization subsystem (e.g., names starting with "visu") and append them to `visu-list` [S1, unofficial].
3.  **Querying:** For each interpreter $k$ in `visu-list`, send a command to query the current minimum and maximum intensity values of the associated image. Append these values to `min` and `max` respectively [S1, unofficial].
    *   Command: `send $k {$curr-img getmin}`
    *   Command: `send $k {$curr-img getmax}`
4.  **Global Computation:** Determine the absolute global minimum ($f_{min}$) and maximum ($f_{max}$) by iterating through the collected `min` and `max` lists.
    *   Initialize $f_{min}$ and $f_{max}$ with the first element of the respective lists.
    *   For each subsequent value $v$ in `min`, if $v < f_{min}$, set $f_{min} = v$.
    *   For each subsequent value $v$ in `max`, if $v > f_{max}$, set $f_{max} = v$ [S1, unofficial].
5.  **Broadcasting:** Iterate through `visu-list` and send the new global range to each instance.
    *   Command: `send $k {$curr-img range} $fmin $fmax` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def synchronize_dynamic_range(visu_list, curr_img):
    """
    Synchronizes the dynamic range of multiple visualization instances.
    
    Args:
        visu_list (list): List of visualization interpreter identifiers.
        curr_img (object): Reference to the current image/volume data object.
        
    Returns:
        None: Updates the range on all instances in visu_list.
    """
    # Step 1 & 2: Filter and prepare lists (Assuming visu_list is pre-filtered 
    # or passed as active visu interpreters)
    # In the source, filtering is done via string matching "visu*"
    # Here we assume the input list contains only relevant interpreters.
    
    local_mins = []
    local_maxs = []
    
    # Step 3: Query min/max from each active instance
    for k in visu_list:
        # Query the minimum value from the instance k
        val_min = send_command(k, f"{{{curr_img} getmin}}")
        local_mins.append(val_min)
        
        # Query the maximum value from the instance k
        val_max = send_command(k, f"{{{curr_img} getmax}}")
        local_maxs.append(val_max)
        
    # Step 4: Compute global bounds
    if not local_mins or not local_maxs:
        return # No instances to sync
        
    fmin = local_mins[0]
    fmax = local_maxs[0]
    
    for v in local_mins:
        if v < fmin:
            fmin = v
            
    for v in local_maxs:
        if v > fmax:
            fmax = v
            
    # Step 5: Broadcast new range to all instances
    for k in visu_list:
        # Set the dynamic range to the computed global bounds
        send_command(k, f"{{{curr_img} range}} {fmin} {fmax}")

def send_command(interpreter, command_str):
    """
    Simulates sending a command to a visualization interpreter and returning the result.
    In the original Tcl implementation, this uses the 'send' command.
    [G] Placeholder for inter-process or inter-widget communication mechanism.
    """
    # Implementation depends on the specific visualization framework (e.g., Tk, VISU)
    # This function would execute the command string on the target interpreter
    # and return the result if it's a query, or None if it's a setter.
    pass
```

**Parameters:**
*   `curr-img`: Variable holding the reference to the currently active image widget or data object [S1, unofficial].
*   `$fmin`, `$fmax`: Computed global minimum and maximum intensity values [S1, unofficial].
*   `$visu-list`: Tcl list containing the names of all active visualization interpreters [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Recalculation Frequency:** The dynamic range must be recalculated and broadcast every time an image or volume slice is updated, as the range likely differs per slice and across systems [S1, unofficial].
*   **System Heterogeneity:** Visualization tools may run on different systems and display images on the same screen, necessitating explicit synchronization rather than relying on local defaults [S1, unofficial].

## Operating characteristics
*   **Complexity:** Time complexity is linear with respect to the number of visualization instances $N$, as it requires $N$ queries and $N$ broadcasts.
*   **Strengths:** Ensures visual consistency across multiple viewers; simplifies comparison of floating-point data; leverages existing hypertools infrastructure (e.g., VISU) [S1, unofficial].
*   **Limitations:** Requires a communication mechanism between visualization instances (e.g., Tcl `send` command); assumes all instances support the `getmin`, `getmax`, and `range` commands [S1, unofficial].
*   **When to use:** When comparing multiple views of the same or similar volume data in real-time; when using hypertools-based visualization frameworks [S1, unofficial].

## Background
The method was introduced in the context of the VISU visualization toolkit, which utilizes the Tk library to build "hypertools" – stand-alone applications that can interact with each other [S1, unofficial]. The concept is attributed to John Ousterhout's definition of hypertools [S1, unofficial]. The specific implementation described uses Tcl scripting to manage inter-widget communication [S1, unofficial].

## Sources
[S1] Hypertools in image and volume visualization, https://digital.library.unt.edu/ark:/67531/metadc672944/, fetched 2026-10-26 (unofficial)
