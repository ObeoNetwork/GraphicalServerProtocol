# Graphical Language Server Protocol

The graphical language server protocol heavily builds on the client-server protocol defined in [Sprotty](https://github.com/theia-ide/sprotty), but adds additional actions to enable editing capabilities, etc. Below, we first introduce the base protocol and action types defined in Sprotty, and subsequently discuss actions that have been added on top of the base protocol to enable editing of graphical models, as well as additional action types.

## Sprotty's Client-server Protocol

The following TypeScript definitions describe the base protocol as defined in the client-server protocol of Sprotty. Note that the following documentation describes the classes defined in Sprotty and largely reuses their class documentation.

### ActionMessage
A general message serves as an envelope carrying an action to be transmited between the client and the server.

```typescript
class ActionMessage {
  /**
   * Used to identify a specific client.
   */
  public readonly clientId: string;

  /**
   * The action to execute.
   */
  public readonly action: Action;
}
```

Action messages processed by the client or the server don't always require a response. Additional information regarding the lifecycle of some action messages can be found in the [Sprotty documentation](https://github.com/theia-ide/sprotty/wiki/Client-Server-Protocol).

Actions contained in action messages are identified by their `kind` attribute. This attribute is required for all actions. Certain actions are meant to be sent from the client to the server or vice versa, while other actions can be sent by both ways, by the client or the server. All actions must extend the default action interface.

```typescript
interface Action {
  /**
   * Unique identifier specifying the kind of action to process.
   */
  readonly kind: string;
}
```

### Sprotty element types

Actions refer to elements in the graphical model via an `elementId`. However, a few actions actually need to transfer the graphical model. Therefore, the graphical model needs to be represented as a serializable `SModelRootSchema`.

```typescript
interface SModelRootSchema extends SModelElementSchema {
    canvasBounds?: Bounds
    revision?: number
}

interface SModelElementSchema {
    type: string
    id: string
    children?: SModelElementSchema[]
}
```


### RequestModelAction

Sent from the client to the server in order to request a graphical model. Usually this is the first message that is sent from the client to the server, so it is also used to initiate the communication. The response is a `SetModelAction` or an `UpdateModelAction`.

```typescript
class RequestModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestModel';

  /**
   * Additional options used to compute the graphical model.
   */
  public readonly options?: { [key: string]: string });
}
```

### SetModelAction

Sent from the server to the client in order to set the model. If a model is already present, it is replaced.

```typescript
class SetModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setModel';

  /**
   * The new graphical model elements.
   */
  public readonly newRoot: SModelRootSchema;
}
```

### UpdateModelAction

Sent from the server to the client in order to update the model. If no model is present yet, this behaves the same as a `SetModelAction`. The transition from the old model to the new one can be animated.

```typescript
class UpdateModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'updateModel';

  /**
   * The new root element of the graphical model.
   */
  public readonly newRoot?: SModelRootSchema;
}
```

### RequestBoundsAction

Sent from the server to the client to request bounds for the given model. The model is rendered invisibly so the bounds can derived from the DOM. The response is a `ComputedBoundsAction`. This hidden rendering round-trip is necessary if the client is responsible for parts of the layout.

```typescript
class RequestBoundsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestBounds';

  /**
   * The model elements to consider to compute the new bounds.
   */
  public readonly newRoot: SModelRootSchema;
}
```

### ComputedBoundsAction

Sent from the client to the server to transmit the result of bounds computation as a response to a `RequestBoundsAction`. If the server is responsible for parts of the layout (see `needsServerLayout` viewer option), it can do so after applying the computed bounds received with this action. Otherwise there is no need to send the computed bounds to the server, so they can be processed locally by the client.

```typescript
class ComputedBoundsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'computedBounds';

  /**
   * The new bounds of the model elements.
   */
  public readonly bounds: ElementAndBounds[];

  /*
   * The revision number.
   */
  public readonly revision?: number;

  /**
   * The new alignment of the model elements.
   */
  public readonly alignments?: ElementAndAlignment[];
}
```

The `ElementAndBounds` type is used to associate new bounds with a model element, which is referenced via its id.

```typescript
class ElementAndBounds {
  /**
   * The identifier of the element.
   */
  public readonly elementId: string;

  /**
   * The new bounds of the element.
   */
  public readonly newBounds: Bounds;
}
```

The bounds are the position (x, y) and dimension (width, height) of an object. As such the `Bounds` type extends both `Point` and `Dimension`.

```typescript
class Bounds extends Point, Dimension {
}
```

A `Point` is composed of the (x,y) coordinates of an object.

```typescript
class Point {
  /**
   * The abscissa of the point.
   */
  public readonly x: number;

  /**
   * The ordinate of the point.
   */
  public readonly y: number;
}
```

The `Dimension` of an object is composed of its width and height.

```typescript
class Dimension {
  /**
   * The width of an element.
   */
  public readonly width: number;

  /**
   * the height of an element.
   */
  public readonly height: number;
}
```

The `ElementAndAlignment` type is used to associate a new alignment with a model element, which is referenced via its id.

```typescript
class ElementAndAlignment {
  /**
   * The identifier of an element.
   */
  public readonly elementId: string;

  /**
   * The new alignment of the element.
   */
  public readonly newAlignment: Point;
}
```

### SetBoundsAction

Sent from the server to the client to update the bounds of some (or all) model elements.

```typescript
class SetBoundsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setBounds';

  /**
   * The elements to update with their new bounds.
   */
  public readonly bounds: ElementAndBounds[];
}
```

### CenterAction

Centers the viewport on the elements with the given identifiers. It changes the scroll setting of the viewport accordingly and resets the zoom to its default. This action can also be created on the client but it can also be sent by the server in order to perform such a viewport change remotely.

```typescript
class CenterAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'center';

  /**
   * The identifier of the elements on which the viewport should be centered.
   */
  public readonly elementIds: string[];

  /**
   * Indicate if the modification of the viewport should be realized with or without support of animations.
   */
  public readonly animate: boolean = true;
}
```

### CollapseExpandAction

Recalculates a model when some specific elements are collapsed or expanded. This action can be sent by a client to the server to let the server compute a new version of the model.

```typescript
class CollapseExpandAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'collapseExpand';
  
  /**
   * The identifier of the elements to expand.
   */
  public readonly expandIds: string[];

  /**
   * The identifier of the elements to collapse.
   */
  public readonly collapseIds: string[];
}
```

### CollapseExpandAllAction

Collapses or expands all elements of the model.

```typescript
 class CollapseExpandAllAction {
  /**
   * The kind of the action.
   */
  public readonly kind = 'collapseExpandAll';
  
  /**
   * Indicates if the elements should be expanded (true) or collapsed (false)
   */
  public readonly expand: boolean = true;
}
```

### RequestExportSvgAction

Used to request the export of the graphical model as an SVG image.

```typescript
class RequestExportSvgAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestExportSvg';
}
```

### ExportSvgAction

Used to export the graphical model as an SVG image.

```typescript
class ExportSvgAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'exportSvg';

  /**
   * The SVG image data.
   */
  public readonly svg: string;
}
```

### FitToScreenAction

Triggered when the user requests the viewer to fit its content to the available drawing area. The resulting fit-to-screen command changes the zoom and scroll settings of the viewport so the model can be shown completely. This action can also be sent from the server to the client in order to perform such a viewport change programmatically.

```typescript
class FitToScreenAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'fit';

  /**
   * The identifier of the elements to fit on screen.
   */
  public readonly elementIds: string[];

  /**
   * The padding that should be visible on the viewport.
   */
  public readonly padding?: number;

  /**
   * The max zoom level authorized.
   */
  public readonly maxZoom?: number;

  /**
   * Indicate if the action should be performed with animation support or not.
   */
  public readonly animate: boolean = true;
}
```

### OpenAction

Used to indicate that an element has been opened. The default behavior will be triggered when an end user double click on an element. It can allow a server to react to this event.

```typescript
class OpenAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'open';

  /**
   * The identifier of the element.
   */
  public readonly elementId: string;
}
```

### RequestPopupModelAction

Triggered when the user hovers the mouse pointer over an element to get a popup with details on that element. This action is sent from the client to the server. The response is a `SetPopupModelAction`.

```typescript
class RequestPopupModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestPopupModel';

  /**
   * The identifier of the elements for which a popup is requested.
   */
  public readonly elementId: string;

  /**
   * The bounds.
   */
  public readonly bounds: Bounds;
}
```

### SetPopupModelAction

Sent from the server to the client to display a popup in response to a `RequestPopupModelAction`. This action can also be used to remove any existing popup by choosing `EMPTY_ROOT` as root element.

```typescript
class SetPopupModelAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setPopupModel';

  /**
   * The model elements composing the popup to display.
   */
  public readonly newRoot: SModelRootSchema;
}
```

### SelectAction

Triggered when the user changes the selection, e.g. by clicking on a selectable element. The action should trigger a change in the `selected` state accordingly, so the elements can be rendered differently. This action is also forwarded to the server, if present, so it may react on the selection change. Furthermore, the server can send such an action to the client in order to change the selection remotely.

```typescript
class SelectAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'elementSelected';

  /**
   * The identifier of the elements to mark as selected.
   */
  public readonly selectedElementsIds: string[] = [];

  /**
   * The identifier of the elements to mark as not selected.
   */
  public readonly deselectedElementsIds: string[] = [];
}
```

### SelectAllAction

Used for selecting or deselecting all elements.

```typescript
class SelectAllAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'allSelected';

  /**
   * If `select` is true, all elements are selected, othewise they are deselected.
   */
  public readonly select: boolean = true;
}
```

### ServerStatusAction

Sent by the server to signal a state change.

```typescript
class ServerStatusAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'serverStatus';

  /**
   * The severity of the status.
   */
  public readonly severity: string;

  /**
   * The message describing the status.
   */
  public readonly message: string;
}
```

## Actions for Editing

### RequestOperationsAction

This action is sent from the client to the server to request the list of available operations. Operations denote requests of the client to _modify_ the model. Note that the model modification is performed on the server. Thus, after the server modified the model, it sends an `UpdateModelAction`.

```typescript
class RequestOperationsAction implements Action {
  public readonly kind = 'requestOperations';
}
```

### SetOperationsAction

The server updates the client on the available operations using a `SetOperationsAction`.

```typescript
class SetOperationsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setOperations';
  /*
   * The list of operations.
   */
  public readonly operations: Operation[]
}

export interface Operation {
    readonly id: string;
    readonly elementTypeId?: string;
    readonly label: string;
    readonly operationKind: string;
    readonly active?: boolean;
}

export namespace OperationKind {
  export const CREATE_NODE = "createNode";
  export const CREATE_CONNECTION = "createConnection";
  export const DELETE_ELEMENT = "delete";
  export const CHANGE_BOUNDS = "changeBoundsOperation";
  export const CHANGE_CONTAINER = "changeContainer";
  export const GENERIC = "generic";
}
```

### GenericOperationAction

The client sends an `GenericOperationAction` to the server, to request the execution of a `generic` operation.

```typescript
class GenericOperationAction implements Action {
  /**
   * The kind of the action.
   */
  readonly kind = 'generic';
  /*
   * The id of the generic operation.
   */
  public readonly id: string;
  /**
   * The context element.
   */
  public readonly elementId?: string;
  /*
   * The context location.
   */
  public readonly location?: Point;
}
```


### CreateNodeOperationAction

The client sends a `CreateNodeOperationAction` to the server, to request the execution of a `createNode` operation.

```typescript
class CreateNodeOperationAction implements Action {
  /**
   * The kind of the action.
   */
  readonly kind = 'createNode';
  /**
   * The type of the element to be created.
   */
  public readonly elementTypeId: string;
  /*
   * The location at which the operation shall be executed.
   */
  public readonly location?: Point;
  /*
   * The container in which the operation shall be executed.
   */
  public readonly containerId?: string;
}
```

### CreateConnectionOperationAction

The client sends a `CreateConnectionOperationAction` to the server, to request the execution of a `createConnection` operation.

```typescript
class CreateConnectionOperationAction implements Action {
  /**
   * The kind of the action.
   */
  readonly kind = 'createConnection';
  /**
   * The type of the element to be created.
   */
  public readonly elementTypeId: string;
  /*
   * The source element.
   */
  public readonly sourceElementId: string;
  /*
   * The target element.
   */
  public readonly targetElementId: string;
}
```

### DeleteElementOperationAction

The client sends a `DeleteElementOperationAction` to the server, to request the execution of a `delete` operation.

```typescript
class DeleteElementOperationAction  implements Action {
  /**
   * The kind of the action.
   */
  readonly kind = 'delete';
  /**
   * The element to be deleted.
   */
  public readonly elementIds: string[];
}
```

### ChangeBoundsOperationAction

The client sends a `ChangeBoundsOperationAction` to the server, to request the execution of a `changeBoundsOperation` operation.

```typescript
class ChangeBoundsOperationAction  implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'changeBounds';

  /**
   * The new bounds of an element.
   */
  public readonly newBounds: ElementAndBounds[];
}
```

### ChangeContainerOperationAction

The client sends a `ChangeContainerOperationAction` to the server, to request the execution of a `changeContainer` operation.

```typescript
class ChangeContainerOperationAction  implements Action {
  /**
   * The kind of the action.
   */
  readonly kind = 'changeContainer';
  /**
   * The element to be changed.
   */
  public readonly elementId: string;
  /**
   * The element container of the changeContainer operation.
   */
  public readonly targetContainerId: string;
  /**
   * The graphical location.
   */
  public readonly location?: Point;
}
```

### RequestTypeHintsAction

Sent from the client to the server in order to request hints on whether certain modifications are allowed for a specific element type. The `RequestTypeHintsAction` is optional, but should usually be among the first messages sent from the client to the server after receiving the model via `RequestModelAction`. The response is a `SetTypeHintsAction`.

```typescript
class RequestTypeHintsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestTypeHints';
}
```

### SetTypeHintsAction

Sent from the server to the client in order to provide hints certain modifications are allowed for a specific element type. These hints specify whether an element can be resized, reloacted and/or deleted. Optionaly, they specifiy a list of element types that can be contained/connected by this element (see also `TypeHint`). The rationale is to avoid a client-server round-trip for user feedback of each synchronous user interaction.

```typescript
class SetTypeHintsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setTypeHints';

  /**
   * The hints for node types
   */
  public readonly nodeHints: NodeTypeHint[];
  
    /**
   * The hints for node types
   */
  public readonly edgeHints: EdgeTypeHint[];
  
}


interface TypeHint{
    /** 
    The id of the element.
    */
    public readonly elementTypeId: string;
    
     /**
     * Specifies whether the element can be relocated.
     */
    public readonly repositionable: boolean;

    /**
     * Specifices wheter the element can be deleted
     */
    public readonly deletable: boolean;

}

interface NodeTypeHint extends TypeHint {
    /**
     * Specifies whether the element can be resized.
     */
    public readonly resizable: boolean;

    /**
     * The types of elements that can be contained by this element (if any)
     */
    public readonly containableElementTypeIds?: string[];
}

interface EdgeTypeHint extends TypeHint {

    /**
     * Specifies whether the routing of this element can be changed.
     */
    public readonly routable: boolean;
    
    /**
     * Allowed source element types for this edge type
     */
    public readonly sourceElementTypeIds: string[];
    
     /**
     * Allowed targe element types for this edge type
     */
    public readonly targetElementTypeIds: string[];

}
```

### ChangeBoundsAction

Triggered when the user changes the position or size of an element. This action concerns only the element's graphical size and position. Whether an element can be resized or repositioned may be specified by the server with a `SetTypeHintsAction` to allow for immediate user feedback before resizing or repositioning.

```typescript
class ChangeBoundsAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'changeBounds';

  /**
   * The new bounds of an element.
   */
  public readonly newBounds: ElementAndBounds[];
}
```

### SaveModelAction

Sent from the client to the server in order to persist the current model state back to the model source.
```typescript
class SaveModelAction implements Action{
  /**
   * The kind of the action.
   */
  public readonly kind = 'saveModel';
}
```

## Additional Actions

### ExecuteServerCommandAction

Sent from the client to the server to invoke the execution of a server command. Server commands may execute arbitrary actions, such as code generation, etc. These actions, however, shouldn't manipulate the model. For such actions, an `ExecuteOperationAction` should be used.

```typescript
class ExecuteServerCommandAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'executeServerCommand';
  /**
   * The id of the server command.
   */
  public readonly commandID: string;
  /**
   * The id of the context element.
   */
  public readonly elementId?: string;
}
```

### RequestLayersAction

Sent from the client to the server in order to request graphical layers. With `RequestModelAction` and `RequestToolsAction`, they are the firsts messages that are sent to the server. The response is a `SetLayersAction`.

```typescript
class RequestLayersAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'requestLayers';
}
```

### SetLayersAction

Sent from the server to the client in order to set the graphical layers. If layers are already presents, they are replaced.

```typescript
class SetLayersAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind = 'setLayers';

  /**
   * The layers of the graphical model.
   */
  public readonly layers: Layer[];
}
```

```typescript
interface Layer {
  /**
   * The identifier.
   */
  readonly id: string;

  /**
   * The name.
   */
  readonly name: string;

  /**
   * Indicates if the layer is currently active or not.
   */
  readonly active: boolean;
}
```

### ToggleLayerAction

Sent from the client to the server in order to toggle a layer. The model, the layers, and the tools may be updated after the changes.

```typescript
class ToggleLayerAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind: string = 'toggleLayer';

  /**
   * The name of the layer.
   */
  public readonly layerName: string;

  /**
   * The new state of the layer.
   */
  public readonly newState: boolean;
}
```
### RequestCommandPaletteActions

This action is sent from the client to the server to request additional command palette actions.

```typescript
class RequestCommandPaletteActions implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind: string = 'requestCommandPaletteActions';
  
 /**
   * The identifier of the elements that are selected.
   */
  public readonly selectedElementsIDs: string[];
}
```

### SetCommandPaletteActions

This action is sent from the server to the client to specify additional command palette actions for a given context. The context is given by the `RequestCommandPaletteActions` in the form of the selected elemens.

```typescript
class SetCommandPaletteActions implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind: string = 'setCommandPaletteActions';

  /**
   * The command palette actions provided by the server.
   */
  public readonly actions: LabeledAction[];
}
```

### IdentifiableRequestAction

This action is sent from the client to the server where the wrapped action is executed. Using the id from the `IdentifiableResponseAction` returned by the server, the client can clearly match the response to the specified request.

```typescript
class IdentifiableRequestAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind: string = 'identifiableRequestAction';

  /**
   * The identifier of the request.
   */
  public readonly id: string;

  /**
   * The wrapped action.
   */
  public readonly action: Action;
}
```

### IdentifiableResponseAction

This action is sent from the server to the client after receiving an `IdentifiableRequestAction`. The server handles the wrapped action as if it was sent directly but wraps the response action to additionally specify the correct request/response id.

```typescript
class IdentifiableRequestAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind: string = 'identifiableResponseAction';

  /**
   * The identifier of the request/response.
   */
  public readonly id: string;

  /**
   * The wrapped action.
   */
  public readonly action: Action;
}
```

### ReconnectConnectionAction

This action is sent from the client to the server to reconnect an existing connection by setting a new source or target element. As opposed to deleting and re-creating an edge, this action has the advantage that the object identity on the server and the properties of the connection are preserved.

```typescript
class ReconnectConnectionAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind: string = 'reconnectConnection';
  
  /**
   * The identifier of the connection.
   */
  public readonly connectionElementId: string;

  /**
   * The identifier of the (new) source element.
   */
  public readonly sourceElementId: string;

  /**
   * The identifier of the (new) target element.
   */
  public readonly targetElementId: string;
}
```

### RerouteConnectionAction

This action is sent from the client to the server to request the change an existing connection route by specifing the new route as an array of routing points.

```typescript
class ReconnectConnectionAction implements Action {
  /**
   * The kind of the action.
   */
  public readonly kind: string = 'rerouteConnection';
  
  /**
   * The identifier of the connection.
   */
  public readonly connectionElementId: string;

  /**
   * The (changed) routing points of the connection.
   */
  public readonly routingPoints: Point[];
}
```
