---
name: sequential-migration
description: Run a full sequential migration from CUBA Platform 7.x to Jmix 2.x Flow UI, wave by wave. Starts with entities and proceeds through fetch plans, business logic, fragments, screens, and security. Use when the user wants to migrate an entire project step by step.
argument-hint: "[source-folder] [target-folder]"
allowed-tools: Read, Grep, Glob, Edit, Write, mcp__jetbrains__list_directory_tree, mcp__jetbrains__get_file_text_by_path, mcp__jetbrains__find_files_by_glob, mcp__jetbrains__find_files_by_name_keyword, mcp__jetbrains__search_in_files_by_text, mcp__jetbrains__search_in_files_by_regex, mcp__jetbrains__create_new_file, mcp__jetbrains__replace_text_in_file, mcp__jetbrains__get_file_problems, mcp__jetbrains__reformat_file, mcp__jetbrains__open_file_in_editor
---

# Sequential Migration

You are a CUBA-to-Jmix migration engineer. You will migrate a CUBA Platform 7.x project to Jmix 2.x (Flow UI) following a strict wave-by-wave sequence.

## Setup

- **Source project:** `source-projects/$0`
- **Target project:** `target-projects/$1`

If arguments are not provided, check `PLAN.md` in the workspace root for source/target paths. If PLAN.md doesn't exist, ask the user to run `/init-migration` first.

## Before starting

1. Read `AGENTS.md` for the overall migration approach
2. Read `migration-rules/010 Common.md` for baseline rules
3. Read `PLAN.md` if it exists — use it to track progress
4. Verify both source and target projects exist and are accessible

## Wave execution

Execute waves in strict order. Before each wave:
1. Read the relevant migration-rules document(s)
2. Tell the user what you're about to do
3. Ask for confirmation before proceeding with AskUserQuestion

After each wave:
1. Update `PLAN.md` — check off completed items
2. Summarize what was migrated and any issues found
3. List any `// TODO: migration` comments left behind

### Wave 1: Entities
**Read:** `020 Entities.md`

For each entity in the CUBA source project:
1. Read the source entity class
2. Apply transformations:
   - Remove CUBA base class (`extends StandardEntity`, `extends BaseUuidEntity`, etc.)
   - Add `@JmixEntity` annotation
   - Add explicit `@Id` field with `@JmixGeneratedValue` (type depends on source base class)
   - Add system fields if entity extended `StandardEntity`: `@Version`, `@CreatedBy`, `@CreatedDate`, `@LastModifiedBy`, `@LastModifiedDate`, `@DeletedBy`, `@DeletedDate`
   - `com.haulmont.cuba.*` imports -> `io.jmix.*` imports
   - `javax.persistence.*` -> `jakarta.persistence.*`
   - `javax.validation.*` -> `jakarta.validation.*`
   - `@NamePattern("%s|name")` -> `@InstanceName` on field or `@InstanceName` + `@DependsOnProperties` method
   - `@MetaProperty(mandatory=true)` -> `@JmixProperty(mandatory=true)`, bare `@MetaProperty` -> remove
   - `java.util.Date` -> appropriate `java.time.*` type (OffsetDateTime for timestamps, LocalDate for dates)
   - Remove `@Temporal` annotations after conversion
   - Remove `serialVersionUID`
   - Remove `@Listeners` (note in TODO for event listener migration)
3. Create the entity in the target project, preserving the same package structure
4. Migrate per-package `messages.properties` to centralized format
5. Migrate related enums (EnumClass implementations — usually only import fixes)

### Wave 2: Fetch Plans
**Read:** `030 Fetch Plans.md`

- Find `views.xml` files in source project (registered in `cuba.viewsConfig` property)
- Convert to Jmix format:
  - `<views>` root element -> `<fetchPlans>`
  - `<view>` element -> `<fetchPlan>`
  - `view="name"` attribute on properties -> `fetchPlan="name"`
  - Namespace: `http://schemas.haulmont.com/cuba/view.xsd` -> `http://jmix.io/schema/core/fetch-plans`
  - Built-in views: `_local` -> `_base`, `_minimal` -> `_instance_name`, `_instance-name` (hyphenated CUBA variant) -> `_instance_name`
- Place in target project and register in `jmix.core.fetch-plans-config` property

### Wave 3: Business Logic
**Read:** `040 Business Logic.md`, `030 Fetch Plans.md`

For each service/bean:
1. `com.haulmont.cuba.*` imports -> appropriate `io.jmix.*`
2. `javax.persistence.*` -> `jakarta.persistence.*`
3. `javax.transaction.Transactional` -> `org.springframework.transaction.annotation.Transactional` (or `jakarta.transaction.Transactional`)
4. `AppBeans.get(ServiceName.class)` -> `@Autowired` injection
5. `@Inject` -> `@Autowired`
6. Remove service `NAME` constants (not needed in Jmix)
7. CUBA `DataManager` API -> Jmix fluent API:
   - `loadList(loadContext)` -> `load(Class).query("...").list()`
   - `commit(commitContext)` -> `save(entity)` or `save(SaveContext)`
   - `.setView("viewName")` -> `.fetchPlan("planName")`
8. CUBA `@Config` interfaces -> Spring `@Value` or `@ConfigurationProperties`
9. Entity listeners (`@Listeners`) -> Spring `@EventListener` / `@TransactionalEventListener` with `EntityChangedEvent` / `EntitySavingEvent`
10. Copy to target preserving package structure

### Wave 4: UI Fragments
**Read:** `050 UI Fragments.md`, `060 UI View Controllers.md`

For each fragment:
1. **Controller:**
   - `@UiController("fragmentId")` + `@UiDescriptor("file.xml")` -> `@FragmentDescriptor("file.xml")`
   - `extends ScreenFragment` -> `extends Fragment<T>` (T = root layout component type, e.g. `FormLayout`)
   - `@Inject` for UI components -> `@ViewComponent`
   - `@Inject` for beans -> `@Autowired`
2. **XML descriptor:**
   - Namespace: `http://schemas.haulmont.com/cuba/screen/fragment.xsd` -> `http://jmix.io/schema/flowui/fragment`
   - `<layout>` -> `<content>`
   - Apply same component mappings as for screens (table->dataGrid, etc.)
3. **Fragment inclusion in host screens:**
   - `<fragment screen="fragmentId"/>` -> `<fragment class="fully.qualified.ClassName"/>`
4. **Lifecycle:**
   - No own `BeforeShowEvent` — use `@Subscribe(target = Target.HOST_CONTROLLER)` for host view events
   - CUBA `Target.PARENT_CONTROLLER` -> Jmix `Target.HOST_CONTROLLER`

### Wave 5: UI Screens -> Views
**Read:** `060 UI View Controllers.md`, `070 UI Data Section.md`, `080 UI Handlers.md`, `090 UI Tables and Actions.md`, `100 UI Dialogs and Notifications.md`, `110 UI UX Rules.md`

This is the biggest wave. For each screen:
1. **Controller migration:**
   - `@UiController` -> `@ViewController`
   - `@UiDescriptor` -> `@ViewDescriptor`
   - Add `@Route(value = "path", layout = MainView.class)` (`com.vaadin.flow.router.Route`)
   - `StandardLookup<T>` -> `StandardListView<T>`
   - `StandardEditor<T>` -> `StandardDetailView<T>`
   - `@Inject` for UI components -> `@ViewComponent`
   - `@Inject` / `@Autowired` for beans -> `@Autowired`
   - All event handlers must be `public`
   - `AfterShowEvent` -> `ReadyEvent`
   - `BeforeCommitChangesEvent` -> `BeforeSaveEvent`
   - Controller ID: `*.browse` -> `*.list`, `*.edit` -> `*.detail`
   - Class name: `*Browse` -> `*ListView`, `*Edit` -> `*DetailView`
   - Package: `screen/` or `web/` -> `view/`

2. **XML descriptor migration:**
   - `<window>` -> `<view>`
   - Namespace: `http://schemas.haulmont.com/cuba/screen/window.xsd` -> `http://jmix.io/schema/flowui/view`
   - `caption` -> `title` (on view) / `label` (on components)
   - Remove `messagesPack` attribute
   - `view="viewName"` in data section -> `fetchPlan="planName"`
   - `<table>`/`<groupTable>` -> `<dataGrid>`
   - `<treeTable>` -> `<treeDataGrid>`
   - Column `id="prop"` -> `property="prop"`
   - `<buttonsPanel>` moves outside the data grid
   - `<simplePagination>` moves outside, add `dataLoader="..."`
   - `<filter>` -> `<genericFilter>`
   - `<groupBox caption="...">` -> `<details summaryText="...">`
   - `<form>` / `<fieldGroup>` -> `<formLayout>`
   - `<lookupField>` -> `<entityComboBox>` or `<comboBox>`
   - `<lookupPickerField>` / `<pickerField>` -> `<entityPicker>`
   - Action types: `create` -> `list_create`, `edit` -> `list_edit`, `view` -> `list_read`, `remove` -> `list_remove`
   - Data section — mostly copy as-is, change `view` -> `fetchPlan`

3. **File naming:** `*-browse.xml` -> `*-list-view.xml`, `*-edit.xml` -> `*-detail-view.xml`

4. **Messages:** Migrate per-package `messages.properties` to centralized format with package prefixes

### Wave 6: Security
**Read:** `010 Common.md`, `120 Security Migration.md`

For each CUBA security definition:
1. `AnnotatedRoleDefinition` classes -> `@ResourceRole` interfaces
2. `@Role` -> `@ResourceRole`
3. `@EntityAccess` -> `@EntityPolicy`
4. `@EntityAttributeAccess` -> `@EntityAttributePolicy`
5. `@ScreenAccess` -> `@ViewPolicy` (update screen IDs: `*.browse` -> `*.list`, `*.edit` -> `*.detail`)
6. `@SpecificAccess` -> `@SpecificPolicy`
7. Access groups with `@JpqlConstraint` -> `@RowLevelRole` with `@JpqlRowLevelPolicy`
8. Create `UiMinimalRole` with `@SpecificPolicy(resources = "ui.loginToUi")` if it doesn't exist

## Progress tracking

After completing each wave, update PLAN.md by marking items as done with `[x]`.

## Important rules

- Never modify files in `source-projects/` — source is read-only
- Do not commit changes automatically — wait for user instruction
- If something can't be migrated cleanly, leave `// TODO: migration <description>` comment
- Use JetBrains MCP tools for all file operations when available
- After creating files, open in editor via `mcp__jetbrains__open_file_in_editor`, wait 2-3 seconds, then run `mcp__jetbrains__get_file_problems` to check for compilation errors
- If a wave is too large, suggest splitting by package (e.g. "Let's migrate screens in `com.company.app.web.customer` first")
