# UI Dialogs & Notifications Migration Rules

## Confirmation Dialog (Option Dialog)

### CUBA
```java
@Inject
private Dialogs dialogs;

public void onRemoveItems() {
    dialogs.createOptionDialog()
            .withCaption("Confirm")
            .withMessage("Are you sure you want to delete selected items?")
            .withActions(
                    new DialogAction(DialogAction.Type.YES, Action.Status.PRIMARY)
                            .withHandler(e -> removeItems()),
                    new DialogAction(DialogAction.Type.NO)
            )
            .show();
}
```

### Jmix
```java
@Autowired
private Dialogs dialogs;

public void onRemoveItems() {
    dialogs.createOptionDialog()
            .withHeader("Confirm")  // withCaption → withHeader
            .withText("Are you sure you want to delete selected items?")  // withMessage → withText
            .withActions(
                    new DialogAction(DialogAction.Type.YES)
                            .withHandler(e -> removeItems()),
                    new DialogAction(DialogAction.Type.NO)
            )
            .open();  // show() → open()
}
```

### Event Handling with Action Suspension

```java
// CUBA
@Subscribe("customersTable.remove")
public void onCustomersTableRemove(Action.ActionPerformedEvent event) {
    dialogs.createOptionDialog()
            .withCaption("Confirm")
            .withMessage("Delete selected customers?")
            .withActions(
                    new DialogAction(DialogAction.Type.YES)
                            .withHandler(e -> {
                                dataManager.remove(customersTable.getSelected());
                                customersDl.load();
                            }),
                    new DialogAction(DialogAction.Type.NO))
            .show();
}

// Jmix - using preventAction/continueAction
@Subscribe("customersDataGrid.remove")
public void onCustomersDataGridRemove(ActionPerformedEvent event) {
    dialogs.createOptionDialog()
            .withHeader("Confirm")
            .withText("Delete selected customers?")
            .withActions(
                    new DialogAction(DialogAction.Type.YES)
                            .withHandler(e -> {
                                // Action will be executed after confirmation
                                event.getSource().execute();
                            }),
                    new DialogAction(DialogAction.Type.NO))
            .open();
    
    // Important: suspend the original action
    event.preventAction();
}
```

## Input Dialog

### CUBA
```java
@Inject
private Dialogs dialogs;

dialogs.createInputDialog(this)
        .withCaption("Enter Values")
        .withParameters(
                InputParameter.stringParameter("name")
                        .withCaption("Name")
                        .withRequired(true),
                InputParameter.doubleParameter("amount")
                        .withCaption("Amount")
                        .withDefaultValue(100.0),
                InputParameter.entityParameter("customer", Customer.class)
                        .withCaption("Customer"),
                InputParameter.enumParameter("status", OrderStatus.class)
                        .withCaption("Status")
        )
        .withActions(DialogActions.OK_CANCEL)
        .withCloseListener(closeEvent -> {
            if (closeEvent.closedWith(DialogOutcome.OK)) {
                String name = closeEvent.getValue("name");
                Double amount = closeEvent.getValue("amount");
                Customer customer = closeEvent.getValue("customer");
                processValues(name, amount, customer);
            }
        })
        .show();
```

### Jmix
```java
@Autowired
private Dialogs dialogs;

// In View controller
dialogs.createInputDialog(this)
        .withHeader("Enter Values")  // withCaption → withHeader
        .withParameters(
                InputParameter.stringParameter("name")
                        .withLabel("Name")  // withCaption → withLabel
                        .withRequired(true),
                InputParameter.doubleParameter("amount")
                        .withLabel("Amount")
                        .withDefaultValue(100.0),
                InputParameter.entityParameter("customer", Customer.class)
                        .withLabel("Customer"),
                InputParameter.enumParameter("status", OrderStatus.class)
                        .withLabel("Status")
        )
        .withActions(DialogActions.OK_CANCEL)
        .withCloseListener(closeEvent -> {
            if (closeEvent.closedWith(DialogOutcome.OK)) {
                String name = closeEvent.getValue("name");
                Double amount = closeEvent.getValue("amount");
                Customer customer = closeEvent.getValue("customer");
                processValues(name, amount, customer);
            }
        })
        .open();  // show() → open()
```

### ⚠️ Important for Fragments
```java
// Jmix - calling from fragment
@Subscribe("createBtn")
public void onCreateBtnClick(ClickEvent<JmixButton> event) {
    // Get parent View
    View<?> parentView = UiComponentUtils.getView(this);
    
    dialogs.createInputDialog(parentView)  // Pass View, not this!
            .withHeader("New Item")
            .withParameters(/* ... */)
            .open();
}
```

## Lookup Dialog

### CUBA
```java
@Inject
private ScreenBuilders screenBuilders;

screenBuilders.lookup(Customer.class, this)
        .withScreenClass(CustomerBrowse.class)
        .withSelectHandler(customers -> {
            for (Customer customer : customers) {
                addCustomerToOrder(customer);
            }
        })
        .withOptions(new MapScreenOptions(
                ParamsMap.of("excludedIds", getExcludedIds())
        ))
        .build()
        .show();
```

### Jmix
```java
@Autowired
private DialogWindows dialogWindows;

dialogWindows.lookup(this, Customer.class)
        .withViewClass(CustomerListView.class)
        .withSelectHandler(customers -> {
            customers.forEach(this::addCustomerToOrder);
        })
        .withViewConfigurer(view -> {
            // Configure view before opening
            view.setExcludedIds(getExcludedIds());
        })
        .open();
```

## Detail/Editor Dialog

### CUBA
```java
@Inject
private ScreenBuilders screenBuilders;

// Creating new entity
screenBuilders.editor(Customer.class, this)
        .newEntity()
        .withScreenClass(CustomerEdit.class)
        .withParentDataContext(dataContext)
        .withInitializer(customer -> {
            customer.setStatus(CustomerStatus.NEW);
            customer.setRegistrationDate(new Date());
        })
        .withAfterCloseListener(afterCloseEvent -> {
            if (afterCloseEvent.closedWith(StandardCloseAction.COMMIT)) {
                Customer saved = afterCloseEvent.getScreen().getEditedEntity();
                customersTable.setSelected(saved);
            }
        })
        .build()
        .show();

// Editing existing entity
screenBuilders.editor(customersTable)
        .withScreenClass(CustomerEdit.class)
        .build()
        .show();
```

### Jmix
```java
@Autowired
private DialogWindows dialogWindows;

// Creating new entity
dialogWindows.detail(this, Customer.class)
        .newEntity()
        .withViewClass(CustomerDetailView.class)
        .withParentDataContext(dataContext)
        .withInitializer(customer -> {
            customer.setStatus(CustomerStatus.NEW);
            customer.setRegistrationDate(LocalDate.now());  // Date → LocalDate
        })
        .withAfterCloseListener(closeEvent -> {
            if (closeEvent.closedWith(StandardOutcome.SAVE)) {  // COMMIT → SAVE
                Customer saved = closeEvent.getView().getEditedEntity();
                customersDataGrid.select(saved);  // setSelected → select
            }
        })
        .open();

// Editing existing entity (with DataGrid)
dialogWindows.detail(customersDataGrid)
        .withViewClass(CustomerDetailView.class)
        .open();
```

## Notifications

### CUBA
```java
@Inject
private Notifications notifications;

// Simple notification
notifications.create()
        .withCaption("Success")
        .withDescription("Customer saved successfully")
        .withType(Notifications.NotificationType.HUMANIZED)
        .show();

// Warning
notifications.create()
        .withCaption("Warning")
        .withDescription("Please check the input data")
        .withType(Notifications.NotificationType.WARNING)
        .show();

// Error
notifications.create()
        .withCaption("Error")
        .withDescription("Failed to save customer")
        .withType(Notifications.NotificationType.ERROR)
        .show();
```

### Jmix
```java
@Autowired
private Notifications notifications;

// Simple notification - short form
notifications.create("Customer saved successfully")
        .withType(Notifications.Type.SUCCESS)
        .show();

// Full form with title
notifications.create("Success", "Customer saved successfully")
        .withType(Notifications.Type.SUCCESS)
        .withPosition(Notification.Position.TOP_END)  // new positioning option
        .withDuration(3000)  // duration in milliseconds
        .show();

// Warning
notifications.create("Warning", "Please check the input data")
        .withType(Notifications.Type.WARNING)
        .show();

// Error with HTML content
notifications.create("Error")
        .withDescription("<strong>Failed to save</strong><br/>Check the log for details")
        .withType(Notifications.Type.ERROR)
        .withContentMode(ContentMode.HTML)  // HTML support
        .show();
```

## Message Dialog (Simple Message)

### CUBA
```java
@Inject
private Dialogs dialogs;

dialogs.createMessageDialog()
        .withCaption("Information")
        .withMessage("Operation completed successfully")
        .withType(Dialogs.MessageType.CONFIRMATION)
        .show();
```

### Jmix
```java
@Autowired
private Dialogs dialogs;

dialogs.createMessageDialog()
        .withHeader("Information")
        .withText("Operation completed successfully")
        .withModal(true)
        .withCloseOnOutsideClick(false)  // new option
        .open();
```

## Custom Content Dialogs

### Jmix - New Feature
```java
// Creating dialog with custom component
VerticalLayout content = new VerticalLayout();
content.add(new H3("Custom Content"));
content.add(new Paragraph("This is a custom dialog content"));

Dialog dialog = new Dialog();
dialog.setHeaderTitle("Custom Dialog");
dialog.add(content);
dialog.getFooter().add(
    new Button("OK", e -> dialog.close()),
    new Button("Cancel", e -> dialog.close())
);
dialog.open();
```

## Common Migration Mistakes

### 1. Wrong Owner for Dialogs from Fragments

```java
// ❌ WRONG - from fragment
dialogs.createInputDialog(this)  // this - fragment, not View!
        .open();

// ✅ CORRECT
View<?> view = UiComponentUtils.getView(this);
dialogs.createInputDialog(view)
        .open();
```

### 2. Forgotten Method Replacements

```java
// ❌ CUBA methods
.withCaption() / .withMessage()
.show()
StandardCloseAction.COMMIT

// ✅ Jmix methods  
.withHeader() / .withText()
.open()
StandardOutcome.SAVE
```

### 3. Incorrect Close Handling

```java
// ❌ WRONG
if (closeEvent.getCloseAction() == COMMIT_ACTION_ID)

// ✅ CORRECT
if (closeEvent.closedWith(StandardOutcome.SAVE))
```

### 4. Date vs LocalDate

```java
// ❌ CUBA
entity.setDate(new Date());

// ✅ Jmix
entity.setDate(LocalDate.now());
entity.setDateTime(LocalDateTime.now());
```
