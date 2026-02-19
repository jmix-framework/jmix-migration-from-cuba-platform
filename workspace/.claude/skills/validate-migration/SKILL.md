---
name: validate-migration
description: Validate migration completeness and correctness from CUBA Platform to Jmix 2.x. Compares source and target projects, checks for leftover CUBA imports, finds TODO comments, and runs build validation. Use after completing migration waves or to check overall progress.
argument-hint: "[wave-name-or-all]"
context: fork
agent: Explore
allowed-tools: Read, Grep, Glob, mcp__jetbrains__list_directory_tree, mcp__jetbrains__get_file_text_by_path, mcp__jetbrains__find_files_by_glob, mcp__jetbrains__find_files_by_name_keyword, mcp__jetbrains__search_in_files_by_text, mcp__jetbrains__search_in_files_by_regex, mcp__jetbrains__get_file_problems
---

# Validate Migration

You are a CUBA-to-Jmix migration QA specialist. You verify that migration from CUBA Platform 7.x to Jmix 2.x is complete and correct.

## Input

`$ARGUMENTS` can be:
- `entities` — validate only entity migration
- `screens` or `views` — validate only UI migration
- `security` — validate only security migration
- `all` or empty — validate everything

## Validation checks

### 1. Completeness check

Compare source and target projects:

**Entities:**
- Find all entity classes in source: look for `extends StandardEntity`, `extends BaseUuidEntity`, `extends BaseLongIdEntity`, `extends BaseIntegerIdEntity`, `extends BaseStringIdEntity`, `extends BaseIdentityIdEntity`, or `@Entity` annotations in `com.haulmont.cuba.*` context
- Verify each has a corresponding `@JmixEntity`-annotated class in target
- Report missing entities

**Screens -> Views:**
- Find all `@UiController` classes in source (package `com.haulmont.cuba.gui.screen`)
- Verify each has a corresponding `@ViewController` class in target
- Report missing views

**Services:**
- Find all `@Service` / `@Component` beans in source (excluding UI controllers and fragments)
- Verify each exists in target
- Report missing services

**Security roles:**
- Find all classes extending `AnnotatedRoleDefinition` (design-time roles) in source
- Verify each has a corresponding `@ResourceRole` interface in target
- Find all classes extending `AnnotatedAccessGroupDefinition` (access groups with constraints) in source
- Verify each has a corresponding `@RowLevelRole` interface in target (or documented exception)
- Check that `UiMinimalRole` with `ui.loginToUi` exists in target

**Fragments:**
- Find all classes extending `ScreenFragment` in source
- Verify each has a corresponding `Fragment<T>` class in target
- Report missing fragments

**Messages:**
- Find all per-package `messages.properties` in source
- Verify entity and screen messages are present in target's centralized `messages.properties`

### 2. Correctness checks

**No leftover CUBA imports in target:**
Search target project for:
- `import com.haulmont.cuba.` (should be migrated to `io.jmix.*`)
- `import com.haulmont.bali.` (should be `io.jmix.core.common.*`)
- `import javax.persistence.` (should be `jakarta.persistence.`)
- `import javax.validation.` (should be `jakarta.validation.`)
- `import javax.annotation.` (should be `jakarta.annotation.`)

**No CUBA patterns in target:**
- `extends StandardEntity` / `extends BaseUuidEntity` / etc. (should be plain POJO with `@JmixEntity`)
- `@NamePattern` (should be `@InstanceName`)
- `@UiController` (should be `@ViewController`)
- `@UiDescriptor` (should be `@ViewDescriptor`)
- `@ScreenAccess` (should be `@ViewPolicy`)
- `extends AnnotatedRoleDefinition` (should be `@ResourceRole` interface)
- `extends AnnotatedAccessGroupDefinition` (should be `@RowLevelRole` interface)
- `extends StandardLookup` (should be `StandardListView`)
- `extends StandardEditor` (should be `StandardDetailView`)
- `extends ScreenFragment` (should be `Fragment<T>`)
- `<window ` in XML (should be `<view `)
- `schemas.haulmont.com` in XML (should be `jmix.io/schema/flowui/`)
- `AppBeans.get(` (should be `@Autowired`)

**Date migration:**
- `java.util.Date` in entity classes (should be `java.time.*`)
- `@Temporal` annotations (should be removed)

**Handler visibility:**
- `protected void on` or `private void on` in view controllers (should be `public`)

**Injection model:**
- `@Inject` in view controllers (should be `@ViewComponent` or `@Autowired`)

**Entity requirements:**
- Entity classes without `@JmixEntity` annotation
- Entity classes without explicit `@Id` field

### 3. TODO audit

Find all `// TODO: migration` comments in target project.
List each with file and line number.

### 4. Build validation

If the target project is open in IntelliJ:
- Run `mcp__jetbrains__get_file_problems` on all migrated Java files (entity classes, view controllers, services, security roles) and XML descriptors
- Prioritize files modified in the most recent wave
- Report any compilation errors

### 5. Generate report

Output a structured report:

```
## Migration Validation Report

### Source: CUBA Platform 7.x
### Target: Jmix 2.x Flow UI

### Completeness
| Category | Source | Target | Missing |
|----------|--------|--------|---------|
| Entities | N | N | list... |
| Views | N | N | list... |
| Services | N | N | list... |
| Roles | N | N | list... |
| Fragments | N | N | list... |

### CUBA Leftovers Found
- [ ] <issue description and location>
...

### Issues Found
- [ ] <issue description and location>
...

### TODOs Remaining
- [ ] <file:line> — <TODO description>
...

### Build Status
- Errors: N
- Warnings: N
- Details: ...

### Overall: X% complete
```

**Calculating overall %:** `(total migrated items across all categories) / (total source items across all categories) * 100`. Items with `// TODO: migration` count as migrated but incomplete — note them separately.

### 6. Update PLAN.md

If PLAN.md exists, add the validation report to it.

## Important

- This skill is read-only — it does NOT modify any files
- Run this after each wave or at any point to check progress
