# Jmix AI-Assisted Migration (CUBA Platform 7.2 → Jmix 2)

This repository is designed to facilitate migration of application projects from CUBA Platform to Jmix with the help of AI agents.

See also [jmix-framework/jmix-migration-from-v1](https://github.com/jmix-framework/jmix-migration-from-v1)

Below we assume you are using Claude Code, but other agents should work as well.

## Migration process

### Setup

- Create `workspace/source-projects` folder and copy or clone the source CUBA project into it, for example `source-projects/myapp`.
- Create project `workspace/target-projects/myapp-jmix` using Jmix Studio, with the same base package as in source project.
- Init git repo and commit all.
- Setup IntelliJ IDEA MCP Server
	- Settings -> Tools -> MCP Server
		- Enable
		- Claude Code -> Auto-Configure
- Open both `myapp` and `myapp-jmix` projects in IntelliJ to be accessible by MCP server.
- Open terminal in `workspace` and run Claude Code.
- `/mcp`, check that `jetbrains` is connected.

### Sequence

We recommend to proceed with migration in the following sequence: entities, fetch plans, business logic, fragments, screens. If the project is large, split each phase to the smaller steps, e.g. by packages.

### Example prompts

After starting the agent and before giving any migration commands, initialize it by the following prompt (replace `myapp` with your app folder name):

```
Your task is to migrate a project from CUBA Platform to Jmix.
The source project is located in the `source-projects/myapp`. The target Jmix project is created in `target-projects/myapp-jmix`.
Read `AGENTS.md` to understand the project structure, migration approach and rules.
```

**Migration prompts:**

```
Migrate all entities.
```

```
Migrate all shared fetch plans.
```

```
Migrate all business logic.
```

```
Migrate fragments if any.
```

```
Migrate all screens in `com.company.myapp.gui.sample` package.
```

```
Migrate all screens.
```

## Claude Code Skills

This project includes a set of Claude Code skills (slash commands) that streamline the migration workflow. Skills are located in `workspace/.claude/skills/` and can be invoked via `/skill-name` in Claude Code.

### `/init-migration [source-project-folder]`

**Starting point for any migration.** Analyzes the CUBA Platform source project and produces a structured `PLAN.md`.

What it does:
- Scans `source-projects/` for source projects
- Collects statistics: entities, screens, services, security roles, fragments, add-ons
- If multiple modules found — asks which one to start with
- Checks if a corresponding target Jmix 2.x project exists in `target-projects/`
- If no target project — warns and asks to generate one via Jmix Studio
- Generates `PLAN.md` with a wave-by-wave migration roadmap

Example:
```
/init-migration myapp
```

### `/sequential-migration [source-folder] [target-folder]`

**Main workflow skill.** Runs a full migration wave by wave, from entities to security.

What it does:
- Follows the 6-wave migration strategy from `AGENTS.md`
- Reads the appropriate migration-rules documents before each wave
- Asks for confirmation before starting each wave
- Tracks progress in `PLAN.md`
- Suggests splitting large waves by package

Waves: Entities → Fetch Plans → Business Logic → Fragments → Screens → Security

Example:
```
/sequential-migration myapp myapp-jmix
```

### `/migrate-entity [entity-name-or-path]`

**Targeted entity migration.** Migrates a single entity class or a package of entities.

What it does:
- Removes CUBA base classes (`StandardEntity`, `BaseUuidEntity`, etc.), adds explicit `@Id` + system fields
- `com.haulmont.cuba.*` → `io.jmix.*`, `javax.persistence.*` → `jakarta.persistence.*`
- `java.util.Date` → `java.time.*` (with proper type selection)
- `@NamePattern` → `@InstanceName` + `@DependsOnProperties`
- Adds `@JmixEntity`, removes `@Listeners`, `serialVersionUID`
- Validates the result via IntelliJ inspections

Example:
```
/migrate-entity Customer
/migrate-entity entity.customer
```

### `/migrate-view [screen-id-or-class-name]`

**Targeted screen-to-view migration.** Converts a CUBA screen into a Jmix 2 Flow UI view.

What it does:
- Migrates Java controller: annotations, base class, event handlers, injections
- Migrates XML descriptor: root element, components, actions, layout
- Renames files and packages (`screen/` or `web/` → `view/`, `*Browse` → `*ListView`)
- Handles component mapping (table → dataGrid, groupBox → details, lookupField → entityComboBox, etc.)

Example:
```
/migrate-view sales_Customer.browse
/migrate-view CustomerBrowse
/migrate-view screen.customer
```

### `/migrate-security [role-class-name]`

**Security roles migration.** Converts CUBA security annotations and adds required permissions.

What it does:
- `AnnotatedRoleDefinition` → `@ResourceRole` interface
- `@ScreenAccess` → `@ViewPolicy`, updates screen IDs (`.browse` → `.list`, `.edit` → `.detail`)
- `AccessGroup` with `@JpqlConstraint` → `@RowLevelRole` with `@JpqlRowLevelPolicy`
- Creates `UiMinimalRole` with mandatory `ui.loginToUi` permission
- Warns about database-stored roles that need SQL migration

Example:
```
/migrate-security CustomersFullAccessRole
/migrate-security all
```

### `/validate-migration [wave-name-or-all]`

**Quality gate.** Verifies migration completeness and correctness.

What it does:
- Compares source vs target: lists missing entities, views, services, roles
- Checks for leftover `com.haulmont.cuba.*` imports and CUBA patterns in target
- Audits all `// TODO: migration` comments
- Runs build validation via IntelliJ
- Generates a structured report with completion percentage

Example:
```
/validate-migration entities
/validate-migration all
```

## Recommended workflow

```
1.  /init-migration myapp            # Analyze and plan
2.  /sequential-migration myapp myapp-jmix   # Start wave-by-wave migration
    # ...or use targeted skills for specific items:
    /migrate-entity Customer
    /migrate-view sales_Customer.browse
3.  /validate-migration all           # Check completeness
4.  /migrate-security all             # Migrate security last
5.  /validate-migration all           # Final check
```

## Key Differences: CUBA Platform vs Jmix 2

| Aspect             | CUBA Platform 7.x                    | Jmix 2.x                            |
|--------------------|---------------------------------------|--------------------------------------|
| **Vaadin**         | Vaadin 8 (Generic UI)                 | Vaadin 24 (Flow UI)                  |
| **Persistence**    | `javax.persistence.*`                 | `jakarta.persistence.*`              |
| **Validation**     | `javax.validation.*`                  | `jakarta.validation.*`               |
| **UI Package**     | `com.haulmont.cuba.gui.*`             | `io.jmix.flowui.*`                   |
| **Screen Package** | `screen` or `web` folder              | `view` folder                        |
| **Browse Screen**  | `StandardLookup`, `*Browse`           | `StandardListView`, `*ListView`      |
| **Edit Screen**    | `StandardEditor`, `*Edit`             | `StandardDetailView`, `*DetailView`  |
| **Tables**         | `table`, `groupTable`                 | `dataGrid`                           |
| **Filter**         | `filter`                              | `genericFilter`                      |
| **XML Root**       | `<window>`                            | `<view>`                             |
| **Base Classes**   | `StandardEntity`, `BaseUuidEntity`    | Plain POJO + `@JmixEntity` + `@Id`  |
| **Instance Name**  | `@NamePattern`                        | `@InstanceName`                      |
| **Security**       | `AnnotatedRoleDefinition` (class)     | `@ResourceRole` (interface)          |
| **Service Lookup** | `AppBeans.get()`                      | `@Autowired`                         |
