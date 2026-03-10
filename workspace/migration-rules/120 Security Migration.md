# Security Migration Rules

## Overview

CUBA Platform 7.x and Jmix 2.x have fundamentally different security models. CUBA uses **classes** extending base definitions; Jmix uses **interfaces** with policy annotations.

## CUBA Security Model

CUBA 7.x has two security mechanisms:

1. **Design-time roles** — classes extending `AnnotatedRoleDefinition` with `@Role` annotation (package `com.haulmont.cuba.security.app.role.annotation`)
2. **Access groups with constraints** — classes extending `AnnotatedAccessGroupDefinition` with `@AccessGroup` annotation (row-level security)

These map to Jmix's two role types:
- `@ResourceRole` (interface) — replaces `AnnotatedRoleDefinition`
- `@RowLevelRole` (interface) — replaces access group constraints

## Design-Time Roles: AnnotatedRoleDefinition → @ResourceRole

### Structural Change

CUBA roles are **classes** with methods returning permission containers. Jmix roles are **interfaces** with `void` methods annotated with policies.

### CUBA Example

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

### Jmix Equivalent

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

### Annotation Mapping

| CUBA (`com.haulmont.cuba.security.app.role.annotation`) | Jmix 2.x | Package |
|---|---|---|
| `@Role(name = "...")` | `@ResourceRole(name = "...", code = "...")` — derive `code` from role name in kebab-case | `io.jmix.security.role.annotation` |
| `@EntityAccess(target, allow)` | `@EntityPolicy(entityClass, actions)` | `io.jmix.security.role.annotation` |
| `@EntityAttributeAccess(target, modify/view)` | `@EntityAttributePolicy(entityClass, attributes, action)` | `io.jmix.security.role.annotation` |
| `@ScreenAccess(allow = {...})` | `@ViewPolicy(viewIds = {...})` | `io.jmix.securityflowui.role.annotation` |
| `@SpecificAccess(permissions = {...})` | `@SpecificPolicy(resources = {...})` | `io.jmix.security.role.annotation` |
| `@ScreenComponentAccess` | `@UiComponentPolicy` (UI Constraints add-on, package `io.jmix.uiconstraints.annotation`) | — |
| *(no CUBA equivalent)* | `@MenuPolicy(menuIds = {...})` | `io.jmix.securityflowui.role.annotation` |

### EntityAccess → EntityPolicy Mapping

| CUBA `EntityOp` | Jmix `EntityPolicyAction` |
|---|---|
| `EntityOp.CREATE` | `EntityPolicyAction.CREATE` |
| `EntityOp.READ` | `EntityPolicyAction.READ` |
| `EntityOp.UPDATE` | `EntityPolicyAction.UPDATE` |
| `EntityOp.DELETE` | `EntityPolicyAction.DELETE` |
| All four combined | `EntityPolicyAction.ALL` |

### Screen ID Updates

Update all screen IDs in `viewIds` and `menuIds`:
- `*.browse` → `*.list`
- `*.edit` → `*.detail`
- `*.lookup` → `*.list`

### @MenuPolicy

For each screen in `@ScreenAccess` that is a list/browse screen, add a corresponding `@MenuPolicy(menuIds = {...})` since CUBA didn't have a separate menu policy.

## Access Groups: AnnotatedAccessGroupDefinition → @RowLevelRole

### CUBA Example

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

### Jmix Equivalent

```java
@RowLevelRole(name = "Active Orders Only", code = "active-orders-only")
public interface ActiveOrdersRole {

    @JpqlRowLevelPolicy(entityClass = Order.class,
        where = "{E}.active = true")
    void order();
}
```

### Constraint Mapping

| CUBA | Jmix 2.x | Package |
|---|---|---|
| `@AccessGroup` | `@RowLevelRole` | `io.jmix.security.role.annotation` |
| `@JpqlConstraint(target, where)` | `@JpqlRowLevelPolicy(entityClass, where)` | `io.jmix.security.role.annotation` |
| `@JpqlConstraint(target, join, where)` | `@JpqlRowLevelPolicy(entityClass, join, where)` | `io.jmix.security.role.annotation` |
| In-memory `@Constraint` method | `@PredicateRowLevelPolicy` | `io.jmix.security.role.annotation` |

### Important Notes

- CUBA access groups form a hierarchy (parent/child). Jmix row-level roles are **flat** — assign multiple roles to a user as needed. The hierarchical structure is lost; document this in a TODO comment.
- **Session attributes** used in CUBA access group constraints have **no equivalent** in Jmix. Add `// TODO: migration - session attributes from access groups need architectural rethinking` if detected.

## UiMinimalRole (Required)

Jmix 2.x requires a role with `ui.loginToUi` permission for users to access the UI. If no such role exists, create one:

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

## Edge Cases

### Wildcard Permissions
If a CUBA role grants permissions to all entities (target = `"*"`), map to:
- `@EntityPolicy(entityName = "*", actions = EntityPolicyAction.ALL)`
- `@EntityAttributePolicy(entityName = "*", attributes = "*", action = EntityAttributePolicyAction.MODIFY)`

For screen wildcards, use `@ViewPolicy(viewIds = "*")` and `@MenuPolicy(menuIds = "*")`.

### Partial Entity Permissions
When a CUBA role grants only some operations (e.g., only READ and UPDATE), use an array:
```java
@EntityPolicy(entityClass = Customer.class, actions = {EntityPolicyAction.READ, EntityPolicyAction.UPDATE})
```

## Database-Stored Roles

If the CUBA project uses database-stored roles (runtime roles, check `cuba.rolesStorageMode` property):
- Runtime role data cannot be automatically migrated
- The CUBA database role tables differ from Jmix's `ResourceRoleEntity` / `RowLevelRoleEntity`
- Add `// TODO: migration - database-stored roles need to be recreated manually in Jmix admin UI`
