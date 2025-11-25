# AI-Assisted Migration: CUBA → Jmix 2.x

A practical guide for iterative migration using AI agents (Claude Code) with wave-based verification and error correction.

## Migration Architecture

```
projectWorkspace/
│
├── migration-docs/
│   └── MigrationRuleSet/              # Wave-specific transformation rules
│       ├── Common Rules.md            # Cross-cutting patterns
│       ├── Entities Rules.md          # Entity annotations & structure
│       ├── Business Logic Rules.md    # Service layer patterns
│       ├── Fetch Plan Rules.md        # Data loading transformations
│       ├── Views - Controller Rules.md # Screen controller patterns
│       ├── Fragments - Controller Rules.md
│       ├── Flow UI - Data Section Rules.md
│       ├── Flow UI - Dialogs & Notifications.md
│       ├── Flow UI - Tables & Actions.md
│       └── Flow UI - UX Rules.md
│
├── srcCubaFolder/                     # Source CUBA project
└── targetJmixFolder/                  # Target Jmix project
    └── AlreadyMigratedModule/         # Reference implementations
```

## Wave Strategy

Migration proceeds in waves, each with its own MigrationRuleSet documentation:

1. **Entities** - Domain model and annotations
2. **Fetch Plans** - Data loading configurations
3. **Services** - Business logic layer
4. **Listeners** - Event handlers
5. **Screens** - UI layer (most complex)
6. **Fragments** - Reusable UI components

## Key LLM Behavior Patterns

### Within-Wave Learning

* **Start of wave**: Agent makes predictable errors despite MigrationRuleSet
* **Mid-wave**: Begins recognizing and self-correcting patterns
* **End of wave**: Automatically applies learned fixes beyond documented rules

### Between Waves: Context Reset

* Learning doesn't transfer between waves (different error types)
* Each wave starts nearly fresh
* This is expected and normal

### Tool Evolution

The agent naturally evolves its tool usage:

* **Early**: Heavy bash usage (`grep`, `find`, `ls`)
* **Middle**: Mixed approach
* **Late**: Primarily IDEA MCP tools (`search`, `get_file_problems`)

## Practical Migration Process

### Initial Setup

```
Hello! Task: migrate CUBA project to Jmix 2.x.
Read MigrationRuleSet/<relevant-wave>.md for transformation patterns.
Source: srcCubaFolder/PROJECT, Target: targetJmixFolder/PROJECT
```

### Wave Execution

#### Phase 1: Bulk Transfer

```
Migrate all entities from CUBA to Jmix.
Use MigrationRuleSet/Entities Rules.md for transformations.
Mark unclear items with // TODO: migration
```

#### Phase 2: Pattern Correction (Batch)

```
Quick scan shows these repeated errors:
- @NamePattern should be @InstanceName
- @MetaProperty not transformed
Fix in ALL files.
```

#### Phase 3: Detailed Verification

```
Open files in editor: Entity1.java, Entity2.java, Entity3.java
Call get_file_problems for each (onlyErrors = false)
Fix all IDEA-reported issues.
```

### Strategic Feedback

**For completeness issues:**

```
You missed screens [list packages]. Transfer everything.
```

**For repeated MigrationRuleSet violations (harsh but effective):**

```
You're making the same error I documented in MigrationRuleSet.
Call get_file_problems before proceeding.
Be more careful and apply previous corrections.
```

**Positive reinforcement:**

```
Good progress. I fixed [X] - notice the pattern and apply it going forward.
```

## Screen Migration Specifics

Screens require special attention:

* Transfer ALL fields and methods from originals
* Use `// TODO: migration` for unclear transformations
* Apply naming conventions: `*ListView`, `*DetailView`
* Entity IDs must follow pattern: `entityName.list`, `entityName.detail`
* Messages go in single `messages.properties` with FQN keys
* `@ViewComponent` for UI injections, `@Autowired` for Spring beans

## Critical Success Factors

1. **MigrationRuleSet is a starting point** - Agent will discover patterns beyond documentation
2. **Negative feedback works** - Increases agent attention to detail
3. **1:1 comparison mandatory** - Only way to ensure completeness
4. **IDEA MCP nearly essential** - Enables self-verification via `get_file_problems`
5. **Group similar files** - Process related screens/services together
6. **Document new patterns** - Update MigrationRuleSet after each wave

## Progress Indicators

✅ Within-wave improvement visible
✅ Transition from bash to IDEA tools
✅ Self-application of fixes
✅ Pattern recognition beyond MigrationRuleSet
✅ Successful localhost:8080 launch

## Final Checklist

* [ ] All design-time errors resolved
* [ ] Clean compilation (no errors/warnings)
* [ ] Runtime verification passed
* [ ] UI matches original 1:1
* [ ] TODOs documented
* [ ] MigrationRuleSet updated with discoveries
* [ ] Patterns captured for next modules

## Key Takeaway

Each wave is a fresh start with predictable initial errors. Success comes from systematic correction, strategic feedback, and allowing the agent to learn patterns within each wave. The structured MigrationRuleSet per wave type ensures consistency while enabling the agent to discover and apply patterns beyond the documentation.