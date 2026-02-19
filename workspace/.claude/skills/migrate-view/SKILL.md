---
name: migrate-view
description: Migrate a CUBA Platform 7.x screen to a Jmix 2.x Flow UI view. Converts XML descriptors (window->view, CUBA namespace->Jmix namespace), controllers (StandardLookup->StandardListView, StandardEditor->StandardDetailView), component mappings (table->dataGrid, filter->genericFilter). Use when migrating a specific screen or set of screens.
argument-hint: "[screen-id-or-class-name]"
allowed-tools: Read, Grep, Glob, Edit, Write, mcp__jetbrains__get_file_text_by_path, mcp__jetbrains__find_files_by_name_keyword, mcp__jetbrains__find_files_by_glob, mcp__jetbrains__search_in_files_by_text, mcp__jetbrains__search_in_files_by_regex, mcp__jetbrains__create_new_file, mcp__jetbrains__replace_text_in_file, mcp__jetbrains__get_file_problems, mcp__jetbrains__reformat_file, mcp__jetbrains__open_file_in_editor
---

# Migrate View

You are a CUBA-to-Jmix UI migration specialist. You convert a CUBA Platform 7.x screen into a Jmix 2.x Flow UI view.

## Before starting

Read these migration rules:
- `migration-rules/010 Common.md`
- `migration-rules/030 Fetch Plans.md`
- `migration-rules/050 UI Fragments.md`
- `migration-rules/060 UI View Controllers.md`
- `migration-rules/070 UI Data Section.md`
- `migration-rules/080 UI Handlers.md`
- `migration-rules/090 UI Tables and Actions.md`
- `migration-rules/100 UI Dialogs and Notifications.md`
- `migration-rules/110 UI UX Rules.md`

Identify source and target projects from `PLAN.md` or ask the user.

## Input

`$ARGUMENTS` can be:
- A screen ID (e.g. `sales_Customer.browse`)
- A controller class name (e.g. `CustomerBrowse`)
- A package (e.g. `screen.customer` or `web.customer`) to migrate all screens in that package

## Migration steps

### 1. Locate the source screen
Find both:
- The Java controller (annotated with `@UiController` from `com.haulmont.cuba.gui.screen`)
- The XML descriptor (referenced in `@UiDescriptor`)

### 2. Migrate the controller

**Class-level changes:**

| CUBA Platform | Jmix 2.x Flow UI |
|---|---|
| `@UiController("Entity.browse")` | `@ViewController("Entity.list")` |
| `@UiController("Entity.edit")` | `@ViewController("Entity.detail")` |
| `@UiDescriptor("entity-browse.xml")` | `@ViewDescriptor("entity-list-view.xml")` |
| `@LookupComponent("entityTable")` | `@LookupComponent("entityDataGrid")` |
| `com.haulmont.cuba.gui.Route` (or no Route) | `@Route(value = "path", layout = MainView.class)` (`com.vaadin.flow.router.Route`). Derive `value` from entity name in plural kebab-case: `Customer` → `"customers"`, `OrderItem` → `"order-items"`. For detail views append `/:id`: `"customers/:id"` |
| `extends StandardLookup<T>` | `extends StandardListView<T>` |
| `extends StandardEditor<T>` | `extends StandardDetailView<T>` |
| `extends Screen` | `extends StandardView` |
| `extends MasterDetailScreen<T>` | No direct equivalent — use separate list + detail views (add TODO) |
| `CustomerBrowse` class name | `CustomerListView` |
| `CustomerEdit` class name | `CustomerDetailView` |
| package `screen.*` or `web.*` | package `view.*` |

**Import changes:**

| CUBA | Jmix 2.x |
|---|---|
| `com.haulmont.cuba.gui.screen.*` | `io.jmix.flowui.view.*` |
| `com.haulmont.cuba.gui.components.*` | `io.jmix.flowui.component.*` (varies by component) |
| `com.haulmont.cuba.gui.model.*` | `io.jmix.flowui.model.*` |
| `com.haulmont.cuba.gui.Notifications` | `io.jmix.flowui.Notifications` |
| `com.haulmont.cuba.gui.Dialogs` | `io.jmix.flowui.Dialogs` |

**Injection changes:**
- `@Inject` for UI components -> `@ViewComponent`
- `@Inject` for Spring beans -> `@Autowired`
- `@Inject MessageBundle` -> `@ViewComponent MessageBundle`
- `@Inject DataContext` -> `@ViewComponent DataContext`
- `@Inject CollectionLoader` -> `@ViewComponent CollectionLoader`

**Event handler changes:**
- All handlers must be `public` (not `protected` or `private`)
- `InitEvent` -> `InitEvent` (same concept)
- `BeforeShowEvent` -> `BeforeShowEvent` (same concept)
- `AfterShowEvent` -> `ReadyEvent`
- `BeforeCommitChangesEvent` -> `BeforeSaveEvent`
- `AfterCommitChangesEvent` -> `AfterSaveEvent`

**Add to views:**
- `@DialogMode(width = "50em")` (or appropriate width) for views opened in dialogs

### 3. Migrate the XML descriptor

**Root element and namespace:**
```xml
<!-- CUBA -->
<window xmlns="http://schemas.haulmont.com/cuba/screen/window.xsd"
        caption="msg://..." messagesPack="com.company.app.screen.customer"
        focusComponent="...">

<!-- Jmix 2.x Flow UI -->
<view xmlns="http://jmix.io/schema/flowui/view"
      title="msg://..." focusComponent="...">
```

Note: Remove `messagesPack` — Jmix uses centralized `messages.properties` with package-prefixed keys.

**Data section:** The `<data>` section structure is very similar. Key changes:
- CUBA `view="viewName"` attribute -> Jmix `fetchPlan="fetchPlanName"` attribute
- CUBA entity names in JPQL queries keep `$` prefix if source used it (e.g. `sales$Customer`)
- Built-in views: `_local` -> `_base`, `_minimal` -> `_instance_name`, `_instance-name` (hyphenated CUBA variant) -> `_instance_name`

**Facets:**
- `<screenSettings id="settingsFacet" auto="true"/>` -> `<settings auto="true"/>`
- Add `<urlQueryParameters>` with `<genericFilter>` and `<pagination>` references

**Layout and component mapping:**

| CUBA | Jmix 2.x Flow UI |
|---|---|
| `<filter>` | `<genericFilter>` |
| `<table>` / `<groupTable>` | `<dataGrid>` |
| `<treeTable>` | `<treeDataGrid>` |
| Column `id="prop"` | Column `property="prop"` |
| `multiselect="true"` | `selectionMode="MULTI"` |
| `<buttonsPanel>` inside table | `<hbox classNames="buttons-panel">` outside dataGrid |
| `<simplePagination/>` inside table | `<simplePagination dataLoader="..."/>` outside, in buttonsPanel endSlot |
| `<groupBox caption="...">` | `<details summaryText="...">` |
| `<form>` / `<fieldGroup>` | `<formLayout>` |
| `<lookupField>` | `<entityComboBox>` or `<comboBox>` |
| `<lookupPickerField>` | `<entityPicker>` |
| `<pickerField>` | `<entityPicker>` |
| `<scrollBox>` | `<scroller>` |
| `<checkBox>` | `<checkbox>` |
| `<label value="...">` | `<span text="...">` |
| `caption="..."` | `label="..."` (on components) / `title="..."` (on view root) |
| `stylename="danger"` | `themeNames="error"` |

**Action types:**
- `type="create"` -> `type="list_create"`
- `type="edit"` -> `type="list_edit"`
- `type="view"` -> `type="list_read"`
- `type="remove"` -> `type="list_remove"`
- `type="excelExport"` -> `type="grdexp_excelExport"`

**File naming:**
- `customer-browse.xml` -> `customer-list-view.xml`
- `customer-edit.xml` -> `customer-detail-view.xml`

### 4. Migrate message bundle

Read the source screen's per-package `messages.properties`. Convert to Jmix centralized format:
- `customerBrowse.caption=Customers` -> `com.company.myapp.view.customer/CustomerListView.title=Customers`
- `customerEdit.caption=Customer` -> `com.company.myapp.view.customer/CustomerDetailView.title=Customer`

Append to the target project's centralized `messages.properties`.

### 5. Create files in target project

Place files under `target-projects/` in `view/` package (not `screen/` or `web/`).

### 6. Validate

- Open file in editor via `mcp__jetbrains__open_file_in_editor`, wait 2-3 seconds
- Run `mcp__jetbrains__get_file_problems` on created Java and XML files
- Fix compilation errors
- Add `// TODO: migration <description>` for unresolvable issues
- Components without direct equivalent: `MasterDetailScreen`, `relatedEntities`, `maskedField`, `groupTable` grouping: leave TODO

### 7. Update PLAN.md

Mark the screen as done in PLAN.md.

## Edge cases

**Screen timers:** CUBA `<timer>` facets have no direct equivalent in Jmix Flow UI. Replace with Vaadin's `UI.getCurrent().access(() -> { ... })` called from a scheduled task, or use `@Push` with `UI.access()`. Add `// TODO: migration - timer needs reworking for Vaadin Push` if encountered.

**MasterDetailScreen:** No direct equivalent in Jmix Flow UI. Create separate list and detail views. Add TODO.

**groupTable grouping:** Use `dataGrid` and note the loss of grouping, or suggest the Grouping Data Grid add-on (Jmix 2.7+).

## Important

- Never modify files in `source-projects/`
- Do not commit changes
- `<data>` section is very similar between CUBA and Jmix — main change is `view` -> `fetchPlan` attribute
- `groupTable` has no built-in equivalent — use `dataGrid` and note the loss of grouping, or suggest the Grouping Data Grid add-on (Jmix 2.7+)
- If a screen references fragments, ensure fragments are migrated first
- CUBA `messagesPack` attribute is removed — Jmix uses centralized messages with package prefixes
