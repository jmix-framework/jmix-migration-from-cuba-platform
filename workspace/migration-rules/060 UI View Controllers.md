# UI View Controller Migration Rules

## Overview

Migrating screen controllers is one of the most complex aspects of moving from CUBA to Jmix 2.x. Controllers change drastically: from inheritance of base classes to composition, from Classic UI to Vaadin Flow, and from simple injections to `@ViewComponent` annotations.

## Basic controller transformation

### Classes and annotations

```java
// CUBA
@UiController("sales_Customer.browse")
@UiDescriptor("customer-browse.xml")
@LookupComponent("customersTable")
public class CustomerBrowse extends StandardLookup<Customer> {
    // ...
}
```

```java
// Jmix
@Route(value = "customers", layout = MainView.class)
@ViewController("Customer.list")
@ViewDescriptor("customer-list-view.xml")
@LookupComponent("customersDataGrid")
@DialogMode(width = "64em")
public class CustomerListView extends StandardListView<Customer> {
    // ...
}
```

#### Critical naming changes

```java
// CUBA: arbitrary IDs are allowed
@UiController("sales_Customer.browse")
@UiController("customerBrowseScreen")
@UiController("my-custom-customer-list")
```

```java
// Jmix: strict convention for standard screens
@ViewController("Customer.list")       // for list views
@ViewController("Customer.detail")     // for detail views
// For modules with a prefix:
@ViewController("sales_Partner.list")
@ViewController("sales_Partner.detail")
```

⚠️ **Critical:** If a controller ID does not follow the `<EntityName>.list` or `<EntityName>.detail` pattern, Jmix will NOT locate the default screens for the entity!

## Dependency injection in controllers

### Injection rules

```java
// CUBA
public class CustomerBrowse extends StandardLookup<Customer> {
    @Inject
    private GroupTable<Customer> customersTable;
    
    @Inject
    private DataManager dataManager;
    
    @Inject
    private Notifications notifications;
    
    @Inject
    private Filter filter;
}
```

```java
// Jmix
public class CustomerListView extends StandardListView<Customer> {
    @ViewComponent  // UI components from the descriptor
    private DataGrid<Customer> customersDataGrid;
    
    @ViewComponent
    private GenericFilter filter;
    
    @Autowired      // Spring beans
    private DataManager dataManager;
    
    @Autowired
    private Notifications notifications;
    
    // Or via constructor injection (preferred)
    private final CustomerService customerService;
    
    public CustomerListView(CustomerService customerService) {
        this.customerService = customerService;
    }
}
```

### View-scoped beans

1. `DataContext` → `@ViewComponent`
2. `MessageBundle` → `@ViewComponent`
3. `BackgroundWorker`, `ViewValidation`, `Downloader`, `Notifications`, `UiComponents`, `UiComponentsGenerator`, `ViewNavigators`, `Actions`, `Dialogs`, `DialogWindows`, `CurrentAuthentication`, `DataComponents` → `@Autowired`

### Rule for choosing the annotation

| Component type        | CUBA injection | Jmix injection              | Scope       |
| --------------------- | -------------- | --------------------------- | ----------- |
| UI component from XML | `@Inject`      | `@ViewComponent`            | UI scope    |
| Spring bean           | `@Inject`      | `@Autowired` or constructor | Application |
| UI-scoped bean        | `@Inject`      | `@ViewComponent`            | UI scope    |
| Data container        | `@Inject`      | `@ViewComponent`            | View scope  |

Use these guidelines to map each type of dependency correctly when migrating your controllers from CUBA to Jmix.

## Working with messages

Use `io.jmix.flowui.view.MessageBundle` instead of `com.haulmont.cuba.gui.screen.MessageBundle` and inject it using the `@ViewComponent` annotation.

## Background tasks

Use `io.jmix.flowui.backgroundtask.BackgroundWorker` instead of `com.haulmont.cuba.gui.executors.BackgroundWorker`. Inject it with `@Autowired` annotation and use its `handle()` method to get `io.jmix.flowui.backgroundtask.BackgroundTaskHandler` instance.

Instead of `com.haulmont.cuba.gui.backgroundwork.BackgroundWorkWindow`, use `createBackgroundTaskDialog` method of `io.jmix.flowui.Dialogs` inteface. For example:
```
BackgroundTask<Integer, Void> task = new EmailTask(selected);
dialogs.createBackgroundTaskDialog(task) 
        .withHeader("Sending reminder emails")
        .withText("Please wait while emails are being sent")
        .withTotal(selected.size())
        .withShowProgressInPercentage(true)
        .withCancelAllowed(true)
        .open();
```

## Main menu

If the source CUBA screen is registered in `web-menu.xml`, create appropriate menu item in target Jmix project's `menu.xml`. For example:

CUBA:
```xml
<menu-config xmlns="http://schemas.haulmont.com/cuba/menu.xsd">
    <menu id="application">
        <item screen="Foo.browse"/>
        <!-- ... -->
```

Jmix:
```xml
<menu-config xmlns="http://jmix.io/schema/flowui/menu">
    <menu id="application" title="msg://com.company.myapp/menu.application.title" opened="true">
        <item view="Foo.list" title="msg://com.company.myapp.view.foo/FooListView.title"/>
        <!-- ... -->
```