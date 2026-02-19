---
name: migrate-entity
description: Migrate a CUBA Platform entity class to Jmix 2.x. Handles base class removal, javax->jakarta imports, Date->java.time conversion, @NamePattern->@InstanceName, @MetaProperty->@JmixProperty, adds @JmixEntity and explicit @Id. Use when migrating a specific entity or a small set of entities.
argument-hint: "[entity-class-name-or-path]"
allowed-tools: Read, Grep, Glob, Edit, Write, mcp__jetbrains__get_file_text_by_path, mcp__jetbrains__find_files_by_name_keyword, mcp__jetbrains__find_files_by_glob, mcp__jetbrains__search_in_files_by_text, mcp__jetbrains__create_new_file, mcp__jetbrains__replace_text_in_file, mcp__jetbrains__get_file_problems, mcp__jetbrains__reformat_file, mcp__jetbrains__open_file_in_editor
---

# Migrate Entity

You are a CUBA-to-Jmix entity migration specialist. You migrate a single entity class from CUBA Platform 7.x to Jmix 2.x.

## Before starting

1. Read `migration-rules/010 Common.md` and `migration-rules/020 Entities.md`
2. Identify source and target projects from `PLAN.md` or ask the user

## Input

`$ARGUMENTS` can be:
- An entity class name (e.g. `Customer`, `Order`, `sales$Customer`)
- A relative path to the entity file in source project
- A package path (e.g. `entity.customer`) to migrate all entities in that package

## Migration steps

### 1. Locate the source entity
Search in `source-projects/` for the entity class. If multiple matches, ask the user to clarify.

### 2. Read and analyze
- Read the full source entity file
- Identify: superclass (StandardEntity, BaseUuidEntity, etc.), fields, associations, enums, embedded types
- Note the `@NamePattern` annotation if present
- Note any `@Listeners`, `@MetaProperty`, `@Extends` annotations
- Check the per-package `messages.properties` for entity/attribute labels

### 3. Apply transformations

**Remove CUBA base class and add explicit fields:**

| CUBA base class | Action |
|---|---|
| `extends StandardEntity` | Remove. Add all system fields (see template below) |
| `extends BaseUuidEntity` | Remove. Add explicit `@Id UUID id` with `@JmixGeneratedValue` |
| `extends BaseLongIdEntity` | Remove. Add explicit `@Id Long id` with `@JmixGeneratedValue` |
| `extends BaseIntegerIdEntity` | Remove. Add explicit `@Id Integer id` with `@JmixGeneratedValue` |
| `extends BaseStringIdEntity` | Remove. Add explicit `@Id String id` (already has `@Id` in source) |
| `extends BaseIdentityIdEntity` | Remove. Add explicit `@Id Long id` with `@GeneratedValue(strategy = GenerationType.IDENTITY)` |

**StandardEntity system fields template** (add these when replacing `extends StandardEntity`):
```java
@Id
@Column(name = "ID")
@JmixGeneratedValue
private UUID id;

@Version
@Column(name = "VERSION", nullable = false)
private Integer version;

@CreatedBy
@Column(name = "CREATED_BY")
private String createdBy;

@CreatedDate
@Column(name = "CREATE_TS")
private OffsetDateTime createdDate;

@LastModifiedBy
@Column(name = "UPDATED_BY")
private String lastModifiedBy;

@LastModifiedDate
@Column(name = "UPDATE_TS")
private OffsetDateTime lastModifiedDate;

@DeletedBy
@Column(name = "DELETED_BY")
private String deletedBy;

@DeletedDate
@Column(name = "DELETE_TS")
private OffsetDateTime deletedDate;
```
Required imports for these fields: `io.jmix.core.entity.annotation.JmixGeneratedValue`, `io.jmix.core.metamodel.annotation.JmixEntity`, `jakarta.persistence.*`, `org.springframework.data.annotation.CreatedBy`, `org.springframework.data.annotation.CreatedDate`, `org.springframework.data.annotation.LastModifiedBy`, `org.springframework.data.annotation.LastModifiedDate`, `io.jmix.core.annotation.DeletedBy`, `io.jmix.core.annotation.DeletedDate`. Generate getters and setters for all system fields.

**Add required annotations:**
- Add `@JmixEntity` annotation to every entity class (import `io.jmix.core.metamodel.annotation.JmixEntity`)
- For JPA entities, `@JmixEntity` should NOT specify the `name` parameter (name comes from `@Entity(name = ...)`)

**Imports:**
- `com.haulmont.cuba.*` -> appropriate `io.jmix.*` imports
- `javax.persistence.*` -> `jakarta.persistence.*`
- `javax.validation.*` -> `jakarta.validation.*`
- `javax.annotation.*` -> `jakarta.annotation.*`
- `com.haulmont.cuba.core.entity.annotation.Listeners` -> remove (use `@EventListener` instead)
- `com.haulmont.cuba.core.entity.annotation.Extends` -> remove (different mechanism in Jmix)

**Instance name:**
- `@NamePattern("%s|name")` (single field) -> add `@InstanceName` on the `name` field, remove `@NamePattern`
- `@NamePattern("%s - %s|firstName,lastName")` (multiple fields) -> create `@InstanceName` + `@DependsOnProperties` method:
  ```java
  @InstanceName
  @DependsOnProperties({"firstName", "lastName"})
  public String getInstanceName(MetadataTools metadataTools) {
      return String.format("%s - %s",
              metadataTools.format(firstName),
              metadataTools.format(lastName));
  }
  ```
- `@NamePattern("#methodName|field1,field2")` (method reference) -> convert to `@InstanceName` + `@DependsOnProperties` method

**Date/Time types:**
- `java.util.Date` with `@Temporal(TemporalType.TIMESTAMP)` -> `java.time.OffsetDateTime`
- `java.util.Date` with `@Temporal(TemporalType.DATE)` -> `java.time.LocalDate`
- `java.util.Date` with `@Temporal(TemporalType.TIME)` -> `java.time.LocalTime`
- `java.util.Date` without `@Temporal` -> `java.time.OffsetDateTime` (default)
- Remove all `@Temporal` annotations after conversion

**MetaProperty -> JmixProperty:**
- `@MetaProperty(mandatory = true)` -> `@JmixProperty(mandatory = true)`
- `@MetaProperty` (no parameters) -> remove (redundant in Jmix)
- `@MetaProperty(related = {"field1", "field2"})` -> `@JmixProperty` + `@DependsOnProperties({"field1", "field2"})`

**Remove obsolete elements:**
- Remove `serialVersionUID` field
- Remove `@MetaClass` annotations (entity name goes in `@Entity(name = ...)`)
- Remove `@Listeners` annotations (note in TODO that listeners need migration to `@EventListener`)
- Remove `@EnableRestore` (no Jmix equivalent)
- Remove `@TrackEditScreenHistory` (no Jmix equivalent)
- Remove `@Extends` (different mechanism in Jmix, add TODO)

**Entity name convention:**
- Keep the original entity name in `@Entity(name = "sales$Customer")` — do NOT change it (database compatibility)
- Table names in `@Table(name = "SALES_CUSTOMER")` — keep unchanged

**Unchanged elements:**
- `EnumClass<T>` implementations — only need import fixes (`io.jmix.core.metamodel.datatype.EnumClass`)
- `@Composition`, `@OnDelete`, `@OnDeleteInverse` — no changes
- `@NumberFormat`, `@CaseConversion` — no changes
- `@Store`, `@DbView` — no changes
- JPA annotations (`@ManyToOne`, `@OneToMany`, `@ManyToMany`, `@JoinColumn`, etc.) — only javax->jakarta

**Embedded entities:**
- Add `@JmixEntity` annotation
- Add `@Embeddable` annotation (from `jakarta.persistence`)
- No `@Id` field needed for embeddables

### 4. Migrate messages

Read the source entity's `messages.properties` file (in the same package as the entity).
Convert to Jmix format and append to the target project's centralized `messages.properties`:

```
# CUBA (per-package messages.properties):
Customer=Customer
Customer.name=Name

# Jmix (centralized messages.properties with package prefix):
com.company.myapp.entity/Customer=Customer
com.company.myapp.entity/Customer.name=Name
```

### 5. Create in target project
- Place entity in the same package structure under `target-projects/`
- Preserve the same file name
- Also migrate any closely related enums or embedded entities

### 6. Validate
- Open file in editor via `mcp__jetbrains__open_file_in_editor`, wait 2-3 seconds
- Run `mcp__jetbrains__get_file_problems` on the created file
- Fix any compilation errors
- If something can't be resolved, add `// TODO: migration <description>`

### 7. Update PLAN.md
If PLAN.md exists, mark this entity as done: `- [x] EntityName`

## Edge cases

**SoftDelete / Versioned interfaces without StandardEntity:**
If an entity extends `BaseUuidEntity` (or similar) but also implements `SoftDelete` and/or `Versioned` interfaces directly, add the corresponding fields explicitly:
- `SoftDelete` → add `@DeletedBy String deletedBy` + `@DeletedDate OffsetDateTime deletedDate` (with column names `DELETED_BY`, `DELETE_TS`)
- `Versioned` → add `@Version Integer version` (with column name `VERSION`)
- `HasUuid` → if not already the `@Id`, add `@JmixGeneratedValue UUID uuid`
These CUBA interfaces do not exist in Jmix — the traits must be expressed through explicit annotated fields.

**AppBeans.get() inside entity classes:**
`AppBeans.get()` used inside entity methods (e.g., transient property getters, `@PostConstruct`) cannot be replaced with `@Autowired` because JPA entities are not Spring beans. Options:
- Preferred: move the logic to a service or event listener
- If unavoidable: use `ApplicationContextProvider.getApplicationContext().getBean(...)` as last resort
- Add `// TODO: migration - AppBeans.get() in entity class needs refactoring to service layer`

**@PostConstruct / @PreDestroy on entities:**
Update `javax.annotation.PostConstruct` → `jakarta.annotation.PostConstruct`. Consider migrating entity lifecycle logic to JPA callbacks (`@PrePersist`, `@PostLoad`) or Jmix's `EntitySavingEvent` / `EntityChangedEvent`.

## Important
- Never modify files in `source-projects/`
- Do not commit changes
- If the entity extends a custom base class (not a standard CUBA base), migrate the base class first
- Entity names and table names must remain the same for database compatibility
