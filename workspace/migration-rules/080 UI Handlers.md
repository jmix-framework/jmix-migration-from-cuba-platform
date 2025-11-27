# UI View Handlers Migration Rules

## Lifecycle Event Mapping (CUBA → Jmix)

When migrating from CUBA to Jmix, screen lifecycle events have been renamed or refactored. Below is a comparison of equivalent events and annotations:

| **CUBA (Screen Controller)**          | **Jmix (View Controller)**                 | **Notes**                                                                                                            |
| ------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `InitEvent` (on screen creation)      | `InitEvent` (on view creation)             | Both fire after UI components are created and injected.                                                              |
| `AfterInitEvent` (post init)          | *(Merged into BeforeShow in Jmix Flow UI)* | Jmix triggers data loaders before showing (no separate AfterInit in Flow UI).                                        |
| `BeforeShowEvent` (before show)       | `BeforeShowEvent` (before show)            | Similar purpose. In Jmix, can fire multiple times if navigating to an already open view.                             |
| `AfterShowEvent` (after screen shown) | **`ReadyEvent`** (after view shown)        | Renamed in Jmix. Fired as the final event when the view is fully ready (equivalent to CUBA’s *onAfterShow* handler). |
| `BeforeCloseEvent` (before close)     | `BeforeCloseEvent` (before close)          | No major change. Allows vetoing close.                                                                               |
| `AfterCloseEvent` (after close)       | `AfterCloseEvent` (after close)            | No major change.                                                                                                     |

**Dependency Injection Changes**: In CUBA screens, `@Inject` was used for both UI components and beans. In Jmix, use `@ViewComponent` to inject UI components (and actions or data containers), and use Spring’s `@Autowired` (or `@Inject`) for backend beans. For example, a CUBA field injection:

```java
// CUBA screen controller
@Inject
private TextField<String> nameField;
```

becomes in Jmix:

```java
// Jmix view controller
@ViewComponent 
private JmixTextField<String> nameField;
```

Jmix still uses the `@Subscribe` annotation for event handlers, but the method names and event classes may differ (see table above for mappings). For overriding framework methods or providing delegates (e.g. custom data loaders or renderers), Jmix uses `@Install` and `@Supply` annotations in the controller. For instance, to provide a custom data loader implementation or a column renderer, you annotate a method with `@Install` or `@Supply` respectively. (In CUBA, similar functionality existed via `@Install` or programmatic assignment.)

## Fragment Lifecycle Events and Annotations in Jmix

**Screen Fragments** (UI components reused inside views) have a different lifecycle in Jmix compared to full views (screens in CUBA). Notably:
- **Fragment Init vs View Init**: In Jmix, when a fragment is created and added to a host view, it triggers its own `Init`/`Ready` events internally, but there is no **separate “BeforeShow” event for fragments**. Instead, the fragment’s controller can listen for the host view’s events if needed. Jmix sends a `Fragment.ReadyEvent` when the fragment and its inner components are fully initialized. This is roughly analogous to the fragment being “after init.” In CUBA, fragments similarly had `InitEvent`/`AfterInitEvent` when created, but no direct “BeforeShow” for fragments – you would use the host screen’s events.
- **Subscribing to Host (Parent) Events**: Because fragments don’t get a `BeforeShowEvent` of their own in Jmix, any logic that needs to run when the fragment’s host screen is shown (or preparing data) should subscribe to the host view’s events. Jmix makes this easy with the `@Subscribe(target = Target.HOST_CONTROLLER)` annotation in the fragment controller. For example, to load fragment data when the host view is about to show, you can do:

```java
@Subscribe(target = Target.HOST_CONTROLLER)
public void onHostBeforeShow(View.BeforeShowEvent event) {
    citiesDl.load(); // load fragment's data loader when host is shown
}
```
In CUBA, the same pattern was achieved with `target = Target.PARENT_CONTROLLER` on the subscriber method. Remember that the `@LoadDataBeforeShow` annotation (which auto-loads data containers in screens) does not apply to fragments, so you must load data manually (as shown above) or rely on the host view to pass in data.
- **Shared DataContext**: Jmix uses a single DataContext for a view and all its fragments. This means that fragments can safely use or even “steal” data containers from the host. In XML, a fragment’s data container can be marked with `provided="true"` to indicate that the host view supplies that container. All entities in fragment and host data containers are merged into the same context, so for example, an editor fragment bound to a host’s `addressDc` container will be editing the same entity instance as the host. Saving the host view will also save changes from the fragment since they share context. This is similar to CUBA’s approach to fragments.
- **Fragment Limitations**: As of Jmix 2.x, fragments are still a newer feature (marked “experimental” in some versions). They cannot define certain UI features like facets inside them. Also, the fragment cannot directly subscribe to navigation events (e.g. a fragment’s controller can’t have an `onBeforeShow()` without specifying a target, as it isn’t a standalone navigable view). You should always attach fragment logic either to its own `ReadyEvent` (for internal init) or to the host’s events (for when the host is shown or closed).


## Subscribing to Parent View Data and Events from a Fragment

Fragments often need to interact with the parent view’s data. Here are common patterns:
- **Provided Data Containers**: As mentioned, you can use the host’s data in a fragment by marking the fragment’s data container as `provided="true"` (matching by id). For example, if the host view has an addressDc container, the fragment XML can declare `<instance id="addressDc" provided="true"/>`. In the fragment’s UI components (fields, tables), you can then bind to `addressDc` as if it were local. At runtime, the framework links it to the host’s container (it will ignore class/attributes on the fragment’s provided container definition). This allows the fragment to display or edit data from the host, and any changes are immediately seen by the host since it’s the same container entity.
- **Listening to Host Data Events**: If the fragment needs to react to changes in the host’s data (for example, the host loaded some collection and the fragment wants to update something), you can subscribe to the host view’s lifecycle events as shown above. Additionally, fragments can obtain the parent controller via `getParentController()` if needed for direct communication, but using events or shared containers is usually cleaner.
- **Fragment Lifecycle Recap**: Typically, you initialize fragment UI in its onInit or onReady handler. For instance, you might populate options in a dropdown inside the fragment when it’s ready. Then, to load actual data, subscribe to the host’s `BeforeShowEvent` (since that’s when the host’s data loaders run). Finally, if the fragment needs to finalize something after host is fully shown, you could use host’s `ReadyEvent` as well. Jmix’s fragment `ReadyEvent` is fired once when the fragment is added (equivalent to after it’s initialized and attached).

## Example: Manual Handling vs Event Subscription (TreeDataGrid Hierarchy)

Certain UI scenarios in Jmix require manual intervention beyond simple event subscriptions or annotations. A prime example is customizing a TreeDataGrid’s hierarchy column with a custom renderer or component. In a standard `DataGrid`, you might supply a custom renderer using `@Supply(subject="renderer")`. But for a hierarchy column (the one displaying tree structure), replacing the renderer directly can break the tree `expand`/`collapse` functionality – as noted by Jmix developers, if you simply attach a generator or renderer to the hierarchy column, “the hierarchy from DataGrid disappears”.

Solution: Remove the default hierarchy column and add a new one programmatically in an init or before-show handler. For example:

```java
@ViewComponent
private TreeDataGrid<Asset> assetsDataGrid;

@Subscribe
public void onInit(InitEvent event) {
    // Remove the existing hierarchy column (assumed to have property "name")
    if (assetsDataGrid.getColumnByKey("name") != null) {
        assetsDataGrid.removeColumn(assetsDataGrid.getColumnByKey("name"));
    }
    // Add a new hierarchy column with custom content
    TreeDataGrid.Column<Asset> nameColumn = assetsDataGrid.addComponentHierarchyColumn(asset -> {
        // Create icon based on asset type (folder or file)
        Icon icon = ComponentUtils.parseIcon(Boolean.TRUE.equals(asset.getIsFolder()) 
                                            ? "vaadin:folder" : "vaadin:file");
        icon.setColor(Boolean.TRUE.equals(asset.getIsFolder()) ? "#ddd009" : "#8383f7");
        icon.setSize("0.9em");
        // Create a label for the asset name
        Span name = new Span(asset.getName());
        // Combine icon and name in a horizontal layout
        HorizontalLayout row = new HorizontalLayout(icon, name);
        row.setAlignItems(FlexComponent.Alignment.CENTER);
        row.setSpacing(true);
        return row;
    })
    .setHeader("Name")
    .setAutoWidth(true)
    .setResizable(true);
    // Ensure the new column is set as the first (hierarchy) column
    assetsDataGrid.setColumnPosition(nameColumn, 0);
}

```


In the code above (based on a real use-case), we intercept the screen’s init and rebuild the `TreeDataGrid`’s columns. This preserves the tree functionality while allowing custom content (icon + text in this case). Simply using an annotation like `@Supply` was insufficient here, because the framework’s default hierarchy renderer needed to be replaced in a controlled way. By manually reconfiguring the columns in an event, we ensure the `TreeDataGrid` is properly set up.

Other cases where manual UI handling is needed: If a UI component doesn’t expose some customization via events or annotations, you may resort to manipulating it in a controller method. Always choose an appropriate lifecycle event (`Init`, `BeforeShow`, `Ready`) where the component is available. For instance, to dynamically modify a complex layout or set focus, you might do it in `onReady()` after all components are attached to the UI. Use event subscriptions for most tasks, but don’t hesitate to directly call UI API methods in the event handlers when necessary.

## Selections

In jmix can do this:

```java
  @Subscribe("tblValues")
    public void onTblValuesSelection(final SelectionEvent<DataGrid<IntegratorClientProductMappingValue>, IntegratorClientProductMappingValue> event) {
        tblConditionalValuesCreate.refreshState();
    }
```

## Conclusion

This guide highlighted the key differences in UI lifecycle between CUBA and Jmix and provided code snippets for adapting event handlers and fragment logic. By mapping CUBA’s screen events to Jmix view events (e.g. using `ReadyEvent` instead of `onAfterShow`) and adjusting annotations (`@Inject` → `@ViewComponent`, etc.), you can migrate UI controllers with minimal friction. Pay special attention to fragment behavior (manual data loading and the single data context) and edge cases like TreeDataGrid hierarchy columns that require a manual approach. With these patterns, developers can confidently refactor CUBA screens into the Jmix architecture while preserving functionality and improving clarity of the UI lifecycle.
