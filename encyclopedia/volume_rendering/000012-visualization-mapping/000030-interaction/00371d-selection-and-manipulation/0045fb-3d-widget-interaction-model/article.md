# 3D Widget Interaction Model

Also called: VTK 3D Widget Model

## Overview
The 3D Widget Interaction Model provides a framework for creating interactive, scene-represented objects that users can directly manipulate to control visualization parameters or data processing pipelines. It solves the problem of bridging user input gestures with specific application logic by decoupling the interaction handling from the camera control. Unlike standard interactor styles which manipulate the view, this model associates a visual representation with the widget, allowing the widget's state to drive downstream data transformations (such as streamlines or clipping planes) via an event-driven observer pattern.

## How it works
The method relies on the VTK command/observer pattern. The core mechanism involves instantiating a widget subclass, placing it in the scene, and binding application logic to its interaction events.

**Inputs and Outputs:**
*   **Input:** User gestures (mouse down, move, up) captured by the `vtkRenderWindowInteractor`.
*   **Output:** Updated widget geometry (e.g., `vtkPolyData`) or state changes that trigger pipeline updates or rendering changes.

**Data Structures:**
*   **Widget Instance:** A subclass of `vtkInteractorObserver` (e.g., `vtkLineWidget`, `vtkBoxWidget2`).
*   **Representation:** A subclass of `vtkWidgetRepresentation` (e.g., `vtkLineRepresentation`, `vtkBoxRepresentation`) that defines the visual geometry and handles the math of the widget's transformation.
*   **Event Callbacks:** Functions or methods registered via `AddObserver()`.

**Steps:**
1.  **Instantiate:** Create the widget object.
2.  **Place:** Position, scale, and orient the widget to be consistent with the target object in the scene. This is done by setting the widget's representation properties.
3.  **Enable:** Configure the widget to respond to user input. By default, VTK widgets are enabled via a "keypress-i" event, but this can be modified to trigger on other events or automatically.
4.  **Observe:** Register callbacks for key events using `AddObserver()`. The critical events are:
    *   `StartInteractionEvent`: Triggered on mouse down.
    *   `InteractionEvent`: Triggered on mouse move.
    *   `EndInteractionEvent`: Triggered on mouse up.
5.  **Process Interaction:** Inside the `InteractionEvent` callback:
    *   Retrieve the current widget geometry using `GetPolyData()` (or specific getters like `GetPlanes()` for box widgets).
    *   Feed this geometry into the application pipeline (e.g., as seed points for streamlines or clipping planes for a volume mapper).
    *   The pipeline updates automatically on the next render cycle because the input data source has been modified.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# 1. Instantiate the widget
widget = vtkLineWidget()

# 2. Place the widget (positioning, scaling, orienting)
# This step sets the initial geometry consistent with the target object
widget.SetPlaceFactor(1.0)
widget.PlaceWidget(bounds) # bounds from the target object

# 3. Enable the widget
# By default, 'keypress-i' enables it, but we can enable it programmatically
widget.On()

# 4. Tie application logic to widget events
def on_start_interaction(obj, event):
    # Handle mouse down
    pass

def on_interaction(obj, event):
    # Handle mouse move
    # Retrieve widget geometry
    poly_data = obj.GetPolyData()
    
    # Feed geometry to a pipeline (e.g., streamline seeding)
    # The pipeline input is modified here
    streamline_filter.SetInputData(poly_data)
    
    # Note: The pipeline does not update immediately here.
    # It updates automatically on the next render call.

def on_end_interaction(obj, event):
    # Handle mouse up
    pass

widget.AddObserver("StartInteractionEvent", on_start_interaction)
widget.AddObserver("InteractionEvent", on_interaction)
widget.AddObserver("EndInteractionEvent", on_end_interaction)

# 5. Render loop
renderer.Render() # Triggers pipeline update if inputs changed
```

**Parameters:**
*   **Enable Event:** Default is "keypress-i". Can be modified via `widget.SetInteractor(interactor)` and custom event binding.
*   **Place Factor:** Controls the scaling of the widget relative to the bounding box of the placed object.

**Edge Cases:**
*   **Multiple Observers:** More than one `vtkInteractorObserver` can watch a `vtkRenderWindow`. Event propagation order matters. Classes like `vtkInteractorEventRecorder` can record and replay these events.
*   **Pipeline Update:** Modifying widget input does not trigger an immediate render. The update occurs on the next `Render()` call.

## Subtopics

### Interactive Box Clipping
A variant using `vtkBoxWidget2` to define a clipping region for volume data. It sets apart from generic widgets by extracting six clipping planes from its representation and applying them directly to the volume mapper.
*   **When to choose:** When the user needs to dynamically clip a volume to an arbitrary axis-aligned box.
*   **Link:** [Interactive Box Clipping](10913-interactive-box-clipping.md) (hypothetical link for the variant).
*   **Steps:**
    1.  Attach a callback to `vtkBoxWidget2` listening for `vtkCommand::InteractionEvent`.
    2.  Cast the caller to `vtkBoxWidget2` and retrieve its representation as `vtkBoxRepresentation`.
    3.  Enable inside-out clipping mode by calling `boxRepresentation->SetInsideOut(1)`.
    4.  Extract the six clipping planes defining the box via `boxRepresentation->GetPlanes(planes_)`.
    5.  Apply the extracted planes to the volume mapper using `volume_->GetMapper()->SetClippingPlanes(planes_)`.

## Operating characteristics
*   **Time Complexity:** Interaction event handling is $O(1)$ for geometry retrieval, but downstream pipeline updates (e.g., streamline generation) depend on the complexity of the specific filter.
*   **Memory Complexity:** Depends on the widget representation and the size of the data structures it generates (e.g., `vtkPolyData` for streamlines).
*   **Strengths:** Decouples interaction logic from rendering logic; provides immediate visual feedback through the widget's representation; integrates seamlessly with VTK's pipeline architecture.
*   **Limitations:** Tightly coupled to VTK's event system; requires manual management of multiple observers if complex interactions are needed.
*   **When to use:** When direct manipulation of visualization parameters (clipping, seeding, orientation) is required via a visual handle in the scene.
*   **When not:** When simple camera control is sufficient (use `vtkInteractorStyle` instead).

## Background
The 3D Widget Interaction Model originates from the Visualization Toolkit (VTK). It was introduced to provide a robust, object-oriented way to handle user interaction with 3D data. The model is based on the command/observer design pattern, allowing widgets to act as both visual elements and interaction handlers. It differs from `vtkInteractorStyle` by having a persistent representation in the scene that can be directly manipulated, whereas `vtkInteractorStyle` is primarily for camera control.

## Sources
[S1] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S2] kpengk/VolumeClipping, https://github.com/kpengk/VolumeClipping, fetched 2026-10-26 (unofficial)
