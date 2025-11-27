# UI Tables & Actions Migration Rules

## 1. Buttons panel (ButtonsPanel → `hbox.buttons-panel`)

### CUBA: built-in `<buttonsPanel>` inside table

```xml
<!-- CUBA -->
<groupTable id="customersTable" width="100%" dataContainer="customersDc">
    <actions>
        <action id="create" type="create"/>
        <action id="edit"   type="edit"/>
        <action id="remove" type="remove"/>
    </actions>

    <buttonsPanel id="buttonsPanel" alwaysVisible="true">
        <button id="createBtn" action="customersTable.create"/>
        <button id="editBtn"   action="customersTable.edit"/>
        <button id="removeBtn" action="customersTable.remove"/>
    </buttonsPanel>

    <columns>
        <column property="name"/>
        <column property="email"/>
    </columns>
</groupTable>
```

### Jmix Flow UI: normal `hbox` with `buttons-panel` style

```xml
<!-- Jmix Flow UI -->
<hbox id="customersButtons" classNames="buttons-panel">
    <button id="createBtn" action="customersDataGrid.create"/>
    <button id="editBtn"   action="customersDataGrid.edit"/>
    <button id="removeBtn" action="customersDataGrid.remove"/>
</hbox>

<dataGrid id="customersDataGrid"
          width="100%"
          dataContainer="customersDc">
    <actions>
        <action id="create" type="list_create"/>
        <action id="edit"   type="list_edit"/>
        <action id="remove" type="list_remove"/>
    </actions>
    <columns>
        <column property="name"/>
        <column property="email"/>
    </columns>
</dataGrid>
```

**Migration idea:**
`<buttonsPanel>` inside table → separate `<hbox classNames="buttons-panel">` above the `dataGrid`. Buttons still bind to actions via `action="dataGridId.actionId"`.

---

## 2. Inline editing in DataGrid

### CUBA: inline editor (Vaadin 8 DataGrid)

```xml
<!-- CUBA -->
<dataGrid id="ordersTable"
          width="100%"
          dataContainer="ordersDc"
          editorEnabled="true"
          editorBuffered="true">

    <columns>
        <column property="code"        editable="true"/>
        <column property="description" editable="true"/>
        <column property="amount"      editable="false"/>
    </columns>

    <buttonsPanel>
        <button id="createBtn" action="ordersTable.create"/>
        <button id="removeBtn" action="ordersTable.remove"/>
    </buttonsPanel>
</dataGrid>
```

Optional custom editor field:

```java
@Subscribe
public void onInit(InitEvent event) {
    ordersTable.addEditorFieldGenerator("amount", column ->
            uiComponents.create(TextField.class));  // custom editor
}
```

### Jmix Flow UI: `editable="true"` + `<editorActionsColumn>`

```xml
<!-- Jmix Flow UI -->
<dataGrid id="conditionalValuesDataGrid"
          width="100%"
          dataContainer="conditionalValuesDc"
          editorBuffered="true">

    <actions>
        <action id="create" type="list_create"/>
        <action id="remove" type="list_remove"/>
    </actions>

    <columns>
        <!-- editable columns -->
        <column property="condition"
                header="Condition"
                editable="true"/>

        <column property="value"
                header="Value"
                editable="true"/>

        <!-- editor buttons column -->
        <editorActionsColumn key="rowEditorActions" width="12em">
            <editButton   icon="PENCIL" text="Edit"/>
            <saveButton   icon="CHECK"  themeNames="success"/>
            <cancelButton icon="CLOSE"  themeNames="error" text="Cancel"/>
        </editorActionsColumn>
    </columns>
</dataGrid>
```

Programmatic start of editing (instead of clicking “Edit”):

```java
// Jmix: start editing the selected row
@Subscribe("conditionalValuesDataGrid.editInline")
public void onEditInlineAction(ActionPerformedEvent event) {
    conditionalValuesDataGrid.getSingleSelectedItemOptional()
            .ifPresent(item -> conditionalValuesDataGrid.getEditor().editItem(item));
}
```

**Migration idea:**

* CUBA: `editorEnabled="true"` on `<dataGrid>`; automatic save/cancel row.
* Jmix: `editable="true"` on each column + optional `<editorActionsColumn>` to show **Edit/Save/Cancel** buttons in a dedicated column. Logic (buffered vs non-buffered) controlled by `editorBuffered`.

---

## 3. `enabledRule` for actions

### CUBA: enable/disable based on selection

```java
// CUBA
@Install(to = "customersTable.remove", subject = "enabledRule")
private boolean customersTableRemoveEnabledRule() {
    Set<Customer> selected = customersTable.getSelected();
    return canBeRemoved(selected);
}
```

### Jmix: same pattern, other naming

```java
// Jmix — action on conditionalValuesDataGrid depends on selection in another grid
@Install(to = "conditionalValuesDataGrid.create", subject = "enabledRule")
private boolean conditionalValuesCreateEnabledRule() {
    // enable only if exactly one row is selected in valuesDataGrid
    return valuesDataGrid.getSelectedItems().size() == 1;
}
```

**Migration idea:**
`@Install(... subject = "enabledRule")` is the same concept. Change component IDs and types, keep rule logic.

---

## 4. Forcing actions to re-check `enabledRule` (`refreshState()`)

By default, `enabledRule` is evaluated on screen init and on events that the action itself knows about (e.g. selection of *its own* table).
If your rule depends on **another** component (e.g. selection in a second DataGrid), you must explicitly refresh the action.

### CUBA: refresh on other table selection

```java
// CUBA
@Subscribe("valuesTable")
public void onValuesTableSelection(Table.SelectionEvent<Value> event) {
    Action createAction = conditionalValuesTable.getActionNN("create");
    createAction.refreshState();  // re-run enabledRule
}
```

### Jmix: same idea with Flow UI DataGrid

```java
// Jmix
@Subscribe("valuesDataGrid")
public void onValuesDataGridSelection(
        SelectionEvent<DataGrid<Value>, Value> event) {
    // also we can INJECT create action via @ViewComponent("table.actionName")
    Action createAction = conditionalValuesDataGrid.getAction("create");
    if (createAction != null) {
        createAction.refreshState();  // triggers conditionalValuesCreateEnabledRule()
    }
}
```

**Migration idea:**

* Rule depends only on the same table → framework usually refreshes automatically.
* Rule depends on *other* components (other table, checkbox, etc.) → subscribe to those events and call `action.refreshState()` manually.

---

Conclusion:

* **ButtonsPanel → `hbox.buttons-panel`**
* **Inline editor → `editable="true"` + `editorActionsColumn`**
* **`enabledRule`** stays the same
* **External conditions** → remember to call `refreshState()` on the action.

## Action classes

Migrate usages of Action classes as follows:

- `com.haulmont.cuba.gui.components.Action` -> `io.jmix.flowui.kit.action.Action`
- `com.haulmont.cuba.gui.components.actions.ItemTrackingAction` -> `io.jmix.flowui.action.list.ItemTrackingAction`


## View standard actions

If the source CUBA browse screen contains these buttons linked with standard actions:

```xml
<!-- cuba -->
<hbox id="lookupActions" spacing="true" visible="false">
    <button action="lookupSelectAction"/>
    <button action="lookupCancelAction"/>
</hbox>
```

then in the Jmix list view add the actions explicitly before `layout` element and refer to them in the buttons:

```xml
<!-- jmix -->
<actions>
    <action id="selectAction" type="lookup_select"/>
    <action id="discardAction" type="lookup_discard"/>
</actions>
<layout padding="false" spacing="true">
    <!-- ... -->
    <hbox id="lookupActions" visible="false">
        <button id="selectBtn" action="selectAction"/>
        <button id="cancelBtn" action="discardAction"/>
    </hbox>
</layout>
```

If the source CUBA edit screen contains these buttons linked with standard actions:

```xml
<!-- cuba -->
<hbox id="editActions" spacing="true">
    <button id="commitAndCloseBtn" action="windowCommitAndClose"/>
    <button id="closeBtn" action="windowClose"/>
</hbox>
```

then in the Jmix detail view add the actions explicitly before `layout` element and refer to them in the buttons:

```xml
<!-- jmix -->
<actions>
    <action id="saveAction" type="detail_saveClose"/>
    <action id="closeAction" type="detail_close"/>
</actions>
<layout padding="false" spacing="true">
    <!-- ... -->
    <hbox id="detailActions">
        <button id="saveAndCloseBtn" action="saveAction"/>
        <button id="closeBtn" action="closeAction"/>
    </hbox>
</layout>
```
