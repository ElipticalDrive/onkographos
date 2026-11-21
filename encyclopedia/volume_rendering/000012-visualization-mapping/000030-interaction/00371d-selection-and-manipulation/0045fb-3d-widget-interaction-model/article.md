# 3D Widget Interaction Model

Also called: VTK 3D Widget Model

## Overview
The 3D Widget Interaction Model provides a framework for creating interactive, scene-represented objects that users can directly manipulate to control visualization parameters or data processing pipelines [S1, book]. It solves the problem of bridging user input gestures with specific application logic by decoupling the interaction handling from the camera control [S1, book]. Unlike standard interactor styles which manipulate the view, this model associates a visual representation with the widget, allowing the widget's state to drive downstream data transformations via an event-driven observer pattern [S1, book].

## How it works
**Inputs and Outputs:**
*   **Input:** User gestures (mouse down, move, up) captured by the render window interactor [S1, book].
*   **Output:** Updated widget geometry (e.g., `vtkPolyData`) or state changes that trigger pipeline updates or rendering changes [S1, book].

**Data Structures:**
*   **Widget Instance:** A subclass of `vtkInteractorObserver` (e.g., `vtkLineWidget`, `vtkBoxWidget2`) [S1, book].
*   **Representation:** A subclass of `vtkWidgetRepresentation` that defines the visual geometry and handles the math of the widget's transformation [S1, book].
*   **Event Callbacks:** Functions or methods registered via `AddObserver()` [S1, book].

**Steps:**
1.  **Instantiate:** Create the widget object [S1, book].
2.  **Place:** Position, scale, and orient the widget to be consistent with the target object in the scene [S1, book].
3.  **Enable:** Configure the widget to respond to user input [S1, book].
4.  **Observe:** Register callbacks for key events using `AddObserver()` [S1, book]. The critical events are: `StartInteractionEvent` (triggered on mouse down), `InteractionEvent` (triggered on mouse move), and `EndInteractionEvent` (triggered on mouse up) [S1, book].
5.  **Process Interaction:** Inside the `InteractionEvent` callback, retrieve the current widget geometry using `GetPolyData()` and feed it into the application pipeline (e.g., as seed points for streamlines) [S1, book]. The pipeline updates automatically on the next render cycle because the input data source has been modified [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_3d_widget(widget_class, target_bounds, pipeline_callback):
    # 1. Instantiate the widget
    widget = widget_class()

    # 2. Place the widget (positioning, scaling, orienting consistent with target object)
    widget.place_widget(target_bounds)

    # 3. Enable the widget
    widget.enable() # Default enables on 'keypress-i', modifiable [S1, book]

    # 4. Tie application logic to widget events
    def on_start(obj, event):
        pass # Triggered on mouse down [S1, book]

    def on_interact(obj, event):
        poly_data = obj.get_poly_data() # Retrieve widget geometry [S1, book]
        pipeline_callback(poly_data) # Feed to application pipeline [S1, book]
        # Pipeline updates automatically on next render [S1, book]

    def on_end(obj, event):
        pass # Triggered on mouse up [S1, book]

    widget.add_observer("StartInteractionEvent", on_start)
    widget.add_observer("InteractionEvent", on_interact)
    widget.add_observer("EndInteractionEvent", on_end)

    return widget

# Usage example
widget = setup_3d_widget(VTKWidget, get_bounds(), lambda data: update_streamlines(data))
render() # Triggers pipeline update if inputs changed [S1, book]
```

**Parameters:**
*   **Enable Event:** Default is "keypress-i" [S1, book]. Can be modified to trigger on other events or automatically [S1, book].
*   **Place Factor:** Not given in the sources.

**Edge Cases:**
*   **Multiple Observers:** More than one `vtkInteractorObserver` can watch a `vtkRenderWindow` at a given time, so event propagation order matters [S1, book]. Classes like `vtkInteractorEventRecorder` can record and replay these events [S1, book].
*   **Pipeline Update:** Modifying widget input does not trigger an immediate render; the update occurs on the next `Render()` call [S1, book].

## Subtopics

### Interactive Box Clipping
A variant using `vtkBoxWidget2` to define a clipping region for volume data. It sets apart from generic widgets by extracting six clipping planes from its representation and applying them directly to the volume mapper. When to choose: When the user needs to dynamically clip a volume to an arbitrary axis-aligned box. Link: [Interactive Box Clipping](10913-interactive-box-clipping.md). Steps: 1. Attach a callback to `vtkBoxWidget2` listening for `vtkCommand::InteractionEvent`. 2. Cast the caller to `vtkBoxWidget2` and retrieve its representation as `vtkBoxRepresentation`. 3. Enable inside-out clipping mode by calling `boxRepresentation.SetInsideOut(1)`. 4. Extract the six clipping planes defining the box via `boxRepresentation.GetPlanes(planes)`. 5. Apply the extracted planes to the volume mapper using `volume.GetMapper().SetClippingPlanes(planes)`.

## Operating characteristics
*   **Time Complexity:** Interaction event handling is $O(1)$ for geometry retrieval, but downstream pipeline updates depend on the complexity of the specific filter [S1, book].
*   **Memory Complexity:** Depends on the widget representation and the size of the data structures it generates (e.g., `vtkPolyData` for streamlines) [S1, book].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Decouples interaction logic from rendering logic; provides immediate visual feedback through the widget's representation; integrates seamlessly with the pipeline architecture [S1, book].
*   **Limitations:** Tightly coupled to the VTK event system; requires manual management of multiple observers if complex interactions are needed [S1, book].
*   **When to use:** When direct manipulation of visualization parameters (clipping, seeding, orientation) is required via a visual handle in the scene [S1, book].
*   **When not:** When simple camera control is sufficient (use `vtkInteractorStyle` instead) [S1, book].

## Background
The 3D Widget Interaction Model originates from the Visualization Toolkit (VTK) [S1, book]. It was introduced to provide a robust, object-oriented way to handle user interaction with 3D data [S1, book]. The model is based on the command/observer design pattern, allowing widgets to act as both visual elements and interaction handlers [S1, book]. It differs from `vtkInteractorStyle` by having a persistent representation in the scene that can be directly manipulated, whereas `vtkInteractorStyle` is primarily for camera control [S1, book].

## Sources
[S1] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S2] kpengk/VolumeClipping, https://github.com/kpengk/VolumeClipping, fetched 2026-10-26 (unofficial)
