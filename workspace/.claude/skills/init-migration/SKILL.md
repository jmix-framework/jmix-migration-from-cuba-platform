---
name: init-migration
description: Initialize migration from CUBA Platform 7.x to Jmix 2.x Flow UI. Analyzes source CUBA project structure, identifies modules, entities, screens, services, security roles, checks for target Jmix project, and builds a migration plan. Use when starting a new migration or when the user says "init migration", "start migration", "analyze project for migration".
argument-hint: "[source-project-folder]"
allowed-tools: Read, Grep, Glob, mcp__jetbrains__list_directory_tree, mcp__jetbrains__get_file_text_by_path, mcp__jetbrains__find_files_by_glob, mcp__jetbrains__find_files_by_name_keyword, mcp__jetbrains__search_in_files_by_text, mcp__jetbrains__create_new_file
---

# Init Migration

You are a CUBA-to-Jmix migration analyst. Your job is to analyze a CUBA Platform 7.x source project and produce a structured migration plan for Jmix 2.x (Flow UI).

## Step 1: Read the migration guidelines

Read `AGENTS.md` in the workspace root to understand the migration approach.
Read `migration-rules/010 Common.md` for baseline requirements.

## Step 2: Discover source projects

Scan `source-projects/` directory. If `$ARGUMENTS` is provided, look specifically for the `source-projects/$ARGUMENTS` project.

For each source project found:
1. Identify the project name, base package, and CUBA version from `build.gradle` (look for `com.haulmont.cuba:cuba-global` dependency version)
2. Count and list all entity classes — look for:
   - Classes extending `StandardEntity`, `BaseUuidEntity`, `BaseLongIdEntity`, `BaseIntegerIdEntity`, `BaseStringIdEntity`, `BaseIdentityIdEntity`
   - Classes annotated with `@Entity` in `com.haulmont.cuba` packages
   - `@NamePattern` annotations (CUBA-specific instance name)
3. Count and list all screens — look for:
   - `@UiController` annotations (package `com.haulmont.cuba.gui.screen`)
   - Classes extending `StandardLookup`, `StandardEditor`, `Screen`, `MasterDetailScreen`
4. Count and list all services (`@Service`, `@Component` beans in service packages)
5. Count and list all security roles:
   - Design-time roles: classes extending `AnnotatedRoleDefinition` with `@Role` annotation (package `com.haulmont.cuba.security.app.role.annotation`)
   - Note if project uses database-stored runtime roles (`cuba.rolesStorageMode` property)
6. Count and list all UI fragments (classes extending `ScreenFragment`)
7. Identify views/fetch plans:
   - `views.xml` files registered in `cuba.viewsConfig` property
   - `@NamePattern` annotations (define `_minimal` view)
8. Check for add-ons: maps, BPM, reports, charts, etc.
9. Identify CUBA-specific patterns requiring special attention:
   - `@Listeners` on entities (entity lifecycle listeners)
   - `@Config` interfaces (CUBA configuration mechanism)
   - `AppBeans.get()` usage (static service lookup)
   - `persistence.xml` / `metadata.xml` files
   - Access groups and constraints (CUBA row-level security)
   - Per-package `messages.properties` files

## Step 3: Check for multiple modules

If there are multiple source projects in `source-projects/`:
- List all of them with their statistics
- Ask the user which module to migrate first using AskUserQuestion
- Record the choice in the plan

## Step 4: Check for target project

For the selected source project, check if a corresponding target exists in `target-projects/`:
- Look for a project with matching or similar name
- If found, verify it's a valid Jmix 2.x project (check `build.gradle` for `io.jmix.flowui`)
- If NOT found:
  - Warn the user that a target Jmix 2.x project is required
  - Recommend: "Please generate a new Jmix 2.x project using Jmix Studio with the same base package (`<detected.base.package>`) and place it in `target-projects/<project-name>`"
  - Stop and wait for the user to create the target project

## Step 5: Generate PLAN.md

Create `PLAN.md` in the workspace root with the following structure:

```markdown
# Migration Plan: <project-name>

## Source Project (CUBA Platform)
- **Location:** source-projects/<name>
- **Base package:** <package>
- **CUBA version:** <version>

## Target Project (Jmix 2.x)
- **Location:** target-projects/<name>
- **Base package:** <package>
- **Jmix version:** <version>

## Project Statistics
| Category | Count | Details |
|----------|-------|---------|
| Entities | N | list... |
| Screens | N | list... |
| Services | N | list... |
| Security Roles | N | list... |
| Fragments | N | list... |
| Views (Fetch Plans) | N | list... |

## CUBA-Specific Patterns Detected
- [ ] Entity listeners (@Listeners): N entities
- [ ] Config interfaces (@Config): N interfaces
- [ ] AppBeans.get() usage: N occurrences
- [ ] Access groups/constraints: N groups
- [ ] Database-stored roles: yes/no
- [ ] Per-package messages.properties: N files

## Add-ons Detected
- [ ] Maps
- [ ] BPM
- [ ] Reports
- [ ] Charts
- [ ] Other: ...

## Migration Waves

### Wave 1: Entities
- [ ] <EntityName1> (extends StandardEntity) — <brief description>
- [ ] <EntityName2> (extends BaseUuidEntity) — <brief description>
...

### Wave 2: Fetch Plans
- [ ] views.xml -> fetch-plans.xml
- [ ] <additional fetch plan files>
...

### Wave 3: Business Logic
- [ ] <ServiceName1>
- [ ] <ServiceName2>
...

### Wave 4: UI Fragments
- [ ] <FragmentName1>
...

### Wave 5: UI Screens -> Views
- [ ] <ScreenName1> (browse) -> <ViewName1> (list)
- [ ] <ScreenName2> (edit) -> <ViewName2> (detail)
...

### Wave 6: Security
- [ ] <RoleName1> (AnnotatedRoleDefinition -> @ResourceRole)
- [ ] <RoleName2>
- [ ] Add UiMinimalRole with ui.loginToUi
- [ ] Migrate access groups to @RowLevelRole (if any)
...

## Notes
- <any special considerations, detected add-ons, potential issues>
```

## Step 6: Present summary

After generating PLAN.md, present a brief summary to the user:
- Total items to migrate per wave
- Any detected risks or blockers (missing target project, unusual add-ons, etc.)
- CUBA-specific patterns that require manual attention (Config interfaces, entity listeners, access groups)
- Recommend starting with `/sequential-migration <source> <target>` or wave-specific skills

## Important

- Never modify any source code during init — this is analysis only
- The source project is CUBA Platform 7.x (not Jmix 1.x) — look for `com.haulmont.cuba.*` packages
- Always use JetBrains MCP tools when available for file operations
