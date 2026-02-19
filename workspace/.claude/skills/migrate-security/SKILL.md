---
name: migrate-security
description: Migrate CUBA Platform 7.x security roles to Jmix 2.x. Converts AnnotatedRoleDefinition to @ResourceRole, @ScreenAccess to @ViewPolicy, access groups to @RowLevelRole, adds UiMinimalRole. Use when migrating security roles and permissions.
argument-hint: "[role-class-name]"
allowed-tools: Read, Grep, Glob, Edit, Write, mcp__jetbrains__get_file_text_by_path, mcp__jetbrains__find_files_by_name_keyword, mcp__jetbrains__find_files_by_glob, mcp__jetbrains__search_in_files_by_text, mcp__jetbrains__create_new_file, mcp__jetbrains__replace_text_in_file, mcp__jetbrains__get_file_problems, mcp__jetbrains__reformat_file
---

# Migrate Security

You are a CUBA-to-Jmix security migration specialist. You convert CUBA Platform 7.x security roles and access groups to Jmix 2.x.

## Before starting

1. Read `migration-rules/010 Common.md` and `migration-rules/120 Security Migration.md`
2. Identify source and target projects from `PLAN.md` or ask the user

## Input

`$ARGUMENTS` can be:
- A role class name (e.g. `CustomersFullAccessRole`)
- `all` to migrate all security roles
- Empty — migrate all roles

## Understanding CUBA security model

CUBA 7.x has two security mechanisms:
1. **Design-time roles** — classes extending `AnnotatedRoleDefinition` with `@Role` annotation (package `com.haulmont.cuba.security.app.role.annotation`)
2. **Access groups with constraints** — classes extending `AnnotatedAccessGroupDefinition` with `@AccessGroup` annotation (row-level security)

These map to Jmix's two role types:
- `@ResourceRole` (interface) — replaces `AnnotatedRoleDefinition`
- `@RowLevelRole` (interface) — replaces access group constraints

## Migration steps

### 1. Find all security definitions in source

Search `source-projects/` for:
- Classes extending `AnnotatedRoleDefinition` with `@Role`
- Classes extending `AnnotatedAccessGroupDefinition` with `@AccessGroup`
- Check `cuba.rolesStorageMode` property for database-stored roles

### 2. Migrate each design-time role (AnnotatedRoleDefinition -> @ResourceRole)

**Structural change:** CUBA roles are classes extending `AnnotatedRoleDefinition` with methods returning permission containers. Jmix roles are interfaces with void methods annotated with policies.

**CUBA role example:**
```java
@Role(name = "Customers Full Access")
public class CustomersFullAccessRole extends AnnotatedRoleDefinition {

    @EntityAccess(target = Customer.class,
        allow = {EntityOp.CREATE, EntityOp.READ, EntityOp.UPDATE, EntityOp.DELETE})
    @Override
    public EntityPermissionsContainer entityPermissions() {
        return super.entityPermissions();
    }

    @EntityAttributeAccess(target = Customer.class, modify = {"name", "email"})
    @Override
    public EntityAttributePermissionsContainer entityAttributePermissions() {
        return super.entityAttributePermissions();
    }

    @ScreenAccess(allow = {"demo_Customer.browse", "demo_Customer.edit"})
    @Override
    public ScreenPermissionsContainer screenPermissions() {
        return super.screenPermissions();
    }

    @SpecificAccess(permissions = {"my.specific.permission"})
    @Override
    public SpecificPermissionsContainer specificPermissions() {
        return super.specificPermissions();
    }
}
```

**Jmix equivalent:**
```java
@ResourceRole(name = "Customers Full Access", code = "customers-full-access")
public interface CustomersFullAccessRole {

    @EntityPolicy(entityClass = Customer.class, actions = EntityPolicyAction.ALL)
    @EntityAttributePolicy(entityClass = Customer.class, attributes = "*", action = EntityAttributePolicyAction.MODIFY)
    @ViewPolicy(viewIds = {"Customer.list", "Customer.detail"})
    @MenuPolicy(menuIds = {"Customer.list"})
    void customer();

    @SpecificPolicy(resources = {"my.specific.permission"})
    void specific();
}
```

**Annotation mapping:**

| CUBA (com.haulmont.cuba.security.app.role.annotation) | Jmix 2.x | Package |
|---|---|---|
| `@Role(name = "...")` | `@ResourceRole(name = "...", code = "...")` — derive `code` from role name in kebab-case: `"Customers Full Access"` → `"customers-full-access"` | `io.jmix.security.role.annotation` |
| `@EntityAccess(target, allow)` | `@EntityPolicy(entityClass, actions)` | `io.jmix.security.role.annotation` |
| `@EntityAttributeAccess(target, modify/view)` | `@EntityAttributePolicy(entityClass, attributes, action)` | `io.jmix.security.role.annotation` |
| `@ScreenAccess(allow = {...})` | `@ViewPolicy(viewIds = {...})` | `io.jmix.securityflowui.role.annotation` |
| `@SpecificAccess(permissions = {...})` | `@SpecificPolicy(resources = {...})` | `io.jmix.security.role.annotation` |
| `@ScreenComponentAccess` | No direct equivalent (add TODO) | — |
| *(no CUBA equivalent)* | `@MenuPolicy(menuIds = {...})` | `io.jmix.securityflowui.role.annotation` |

**EntityAccess -> EntityPolicy mapping:**

| CUBA `EntityOp` | Jmix `EntityPolicyAction` |
|---|---|
| `EntityOp.CREATE` | `EntityPolicyAction.CREATE` |
| `EntityOp.READ` | `EntityPolicyAction.READ` |
| `EntityOp.UPDATE` | `EntityPolicyAction.UPDATE` |
| `EntityOp.DELETE` | `EntityPolicyAction.DELETE` |
| All four combined | `EntityPolicyAction.ALL` |

**Update all screen IDs in `viewIds` and `menuIds`:**
- `*.browse` -> `*.list`
- `*.edit` -> `*.detail`
- `*.lookup` -> `*.list`

**Add `@MenuPolicy`:** For each screen in `@ScreenAccess` that is a list/browse screen, add a corresponding `@MenuPolicy(menuIds = {...})` since CUBA didn't have a separate menu policy.

### 3. Migrate access groups with constraints (-> @RowLevelRole)

**CUBA access group with JPQL constraint:**
```java
@AccessGroup(name = "Active Orders Only", parent = RootGroup.class)
public class ActiveOrdersGroup extends AnnotatedAccessGroupDefinition {

    @JpqlConstraint(target = Order.class, where = "{E}.active = true")
    @Override
    public ConstraintsContainer accessConstraints() {
        return super.accessConstraints();
    }
}
```

**Jmix equivalent:**
```java
@RowLevelRole(name = "Active Orders Only", code = "active-orders-only")
public interface ActiveOrdersRole {

    @JpqlRowLevelPolicy(entityClass = Order.class,
        where = "{E}.active = true")
    void order();
}
```

**Constraint mapping:**

| CUBA | Jmix 2.x | Package |
|---|---|---|
| `@AccessGroup` | `@RowLevelRole` | `io.jmix.security.role.annotation` |
| `@JpqlConstraint(target, where)` | `@JpqlRowLevelPolicy(entityClass, where)` | `io.jmix.security.role.annotation` |
| `@JpqlConstraint(target, join, where)` | `@JpqlRowLevelPolicy(entityClass, join, where)` | `io.jmix.security.role.annotation` |
| In-memory `@Constraint` method | `@PredicateRowLevelPolicy` | `io.jmix.security.role.annotation` |

**Important:** CUBA access groups form a hierarchy (parent/child). Jmix row-level roles are flat — assign multiple roles to a user as needed. The hierarchical structure is lost; document this in a TODO comment.

**Session attributes** used in CUBA access group constraints have **no equivalent** in Jmix. Add `// TODO: migration - session attributes from access groups need architectural rethinking` if detected.

### 4. Create UiMinimalRole

If no role with `@SpecificPolicy(resources = "ui.loginToUi")` exists in the target project, create one:

```java
package <base.package>.security;

import io.jmix.security.model.SecurityScope;
import io.jmix.security.role.annotation.ResourceRole;
import io.jmix.security.role.annotation.SpecificPolicy;
import io.jmix.securityflowui.role.annotation.ViewPolicy;

@ResourceRole(name = "UI: minimal access", code = UiMinimalRole.CODE, scope = SecurityScope.UI)
public interface UiMinimalRole {
    String CODE = "ui-minimal";

    @ViewPolicy(viewIds = "MainView")
    @SpecificPolicy(resources = "ui.loginToUi")
    void main();
}
```

### 5. Check for database-stored roles

If the CUBA project uses database-stored roles (runtime roles), note that:
- Runtime role data cannot be automatically migrated
- The CUBA database role tables differ from Jmix's `ResourceRoleEntity` / `RowLevelRoleEntity`
- Add `// TODO: migration - database-stored roles need to be recreated manually in Jmix admin UI`

### 6. Validate

- Run `mcp__jetbrains__get_file_problems` on created files
- Verify all referenced view IDs match the actually migrated views
- Fix compilation errors

### 7. Update PLAN.md

Mark migrated roles as done.

## Edge cases

**Wildcard permissions:** If a CUBA role grants permissions to all entities (target = `"*"`), map to:
- `@EntityPolicy(entityClass = Object.class, actions = EntityPolicyAction.ALL)`
- `@EntityAttributePolicy(entityClass = Object.class, attributes = "*", action = EntityAttributePolicyAction.MODIFY)`
For screen wildcards, Jmix does not support wildcard view policies — list all known view IDs explicitly in `@ViewPolicy`.

**Partial entity permissions:** When a CUBA role grants only some operations (e.g., only READ + UPDATE), emit individual actions:
- `@EntityPolicy(entityClass = Customer.class, actions = {EntityPolicyAction.READ, EntityPolicyAction.UPDATE})`

## Important

- Never modify files in `source-projects/`
- Do not commit changes
- `ui.loginToUi` is mandatory in Jmix 2.x — without it users cannot log in
- Entity policies (`@EntityPolicy`, `@EntityAttributePolicy`) are unchanged between CUBA->Jmix conceptually but use different annotations
- CUBA roles are **classes**, Jmix roles are **interfaces** — this is a structural change, not just a rename
